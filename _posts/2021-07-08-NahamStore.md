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
  It is the edit page for the subdomain `marketing.nahamstore.thm`, we can modify the page here. At this point, possible path is we can add some php code to open a reverse shell to our computer


