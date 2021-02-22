---
layout: post
title:  "Tenable CTF writeups"
date:   2021-02-22 19:59:36 +0200
categories: ctf
---

# Tenable CTF

These are the challenges from [Tenable CTF](https://tenable.ctfd.io/challenges) that I personnaly solved.

## Random encryption fixed

```python=
import random

rands = [[249, 182, 79],[136, 198, 95],[159, 167, 6],
[223, 136, 101],[66, 27, 77],[213, 234, 239],[25, 36, 53],
[89, 113, 149],[65, 127, 119],[50, 63, 147],
[204, 189, 228],[228, 229, 4],[64, 12, 191],
[65, 176, 96],[185, 52, 207],[37, 24, 110],
[62, 213, 244],[141, 59, 81],[166, 50, 189],
[228, 5, 16],[59, 42, 251],[180, 239, 144],
[13, 209, 132]]
res = [184, 161, 235, 97, 140, 111, 84, 182, 162, 135, 76, 10, 69, 246, 195, 152, 133, 88, 229, 104, 111, 22, 39]
seeds = [9925, 8861, 5738, 1649, 2696, 6926, 1839, 7825, 6434, 9699, 227, 7379, 9024, 817, 4022, 7129, 1096, 4149, 6147, 2966, 1027, 4350, 4272]

for i in range(0,len(res)):
    random.seed(seeds[i])
    rands = []
    for j in range(0,4):
        rands.append(random.randint(0,255))
    print (chr(res[i] ^ rands[i%4]),end='')           
print("")
```

## Short and sweet

```python=
import sys

answer = []
def main():
    lines = sys.stdin.readlines()
    for line in lines:
        ints = line.split(" ")
        for i in ints:
            answer.append(int(i) % 2 == 0)
    print (answer)
main()
```

## Hello ${name}

```c=
#include <stdio.h>
int main()
{
   char name[256];
   fgets(name,256,stdin);
   printf ("Hello %s\n",name);
   return 0;
}
```

## Print N Lowest

Sorting algorithm grabbed from: https://www.geeksforgeeks.org/c-program-to-sort-an-array-in-ascending-order/

```cpp=
#include <iostream>

void swap(int* xp, int* yp) 
{ 
    int temp = *xp; 
    *xp = *yp; 
    *yp = temp; 
} 
  
// Function to perform Selection Sort 
void selectionSort(int arr[], int n) 
{ 
    int i, j, min_idx; 
  
    // One by one move boundary of unsorted subarray 
    for (i = 0; i < n - 1; i++) { 
  
        // Find the minimum element in unsorted array 
        min_idx = i; 
        for (j = i + 1; j < n; j++) 
            if (arr[j] < arr[min_idx]) 
                min_idx = j; 
  
        // Swap the found minimum element 
        // with the first element 
        swap(&arr[min_idx], &arr[i]); 
    } 
} 

void PrintNLowestNumbers(int arr[], unsigned int length, unsigned short nLowest)
{
	selectionSort(arr,length);
	for (int i=0;i<nLowest;i++) {
		printf("%d ",arr[i]);
	}
}

int main()
{
	char input[0x100];
	int integerList[0x100];
	unsigned int length;
	unsigned short nLowest;
	std::cin >> nLowest;
	std::cin >> length;
	for (int i=0;i<length;i++)
		 std::cin >> integerList[i];
	PrintNLowestNumbers(integerList, length, nLowest);
}

```

## Spring MVC 1

```java=
	@GetMapping("/main")
        public ModelAndView getMain() {
		ModelAndView modelAndView = new ModelAndView("flag");
                modelAndView.addObject("flag", flags.getFlag("spring_mvc_1"));	// get main
                return modelAndView;
        }
```
Based on this the request to get the flag is http://challenges.ctfd.io:30542/main

## Spring MVC 2 & 3

```java=
	@PostMapping("/main")
        public String postMain(@RequestParam(name="magicWord", required=false, defaultValue="") String magicWord, Model model) {
		if (magicWord.equals("please"))
			model.addAttribute("flag", flags.getFlag("spring_mvc_3"));	// post main param 
		else
                	model.addAttribute("flag", flags.getFlag("spring_mvc_2"));	// post main
                return "flag";
        }

```
These are POST requests so use curl:
```
curl -X POST 'http://challenges.ctfd.io:30542/main?magicWord=wnow'
<!DOCTYPE HTML>
<html>
<head>
    <title>Tenable CTF: Spring MVC</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
	<p >flag{flag2_de3981}</p>
</body>
</html>
```

```
curl -X POST 'http://challenges.ctfd.io:30542/main?magicWord=please'
<!DOCTYPE HTML>
<html>
<head>
    <title>Tenable CTF: Spring MVC</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
	<p >flag{flag3_0d431e}</p>
</body>
</html>
```

## Spring MVC 4
```java=
@PostMapping(path = "/main", consumes = "application/json")
	public String postMainJson(Model model) {
                model.addAttribute("flag", flags.getFlag("spring_mvc_4"));	// post main flag json
                return "flag";
        }
```
```
curl -X POST -H "Content-Type: application/json"  'http://challenges.ctfd.io:30542/main'
<!DOCTYPE HTML>
<html>
<head>
    <title>Tenable CTF: Spring MVC</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
	<p >flag{flag4_695954}</p>
</body>
</html>
```

## Spring MVC 5

```java=
	@RequestMapping(path = "/main", method = RequestMethod.OPTIONS)
        public String optionsMain(Model model) {
                model.addAttribute("flag", flags.getFlag("spring_mvc_5"));	// options main
                return "flag";
        }
```

```
curl -X OPTIONS 'http://challenges.ctfd.io:30542/main'        
<!DOCTYPE HTML>
<html>
<head>
    <title>Tenable CTF: Spring MVC</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
	<p >flag{flag5_70102b}</p>
</body>
</html>

```

## Spring MVC 6

```java=
@RequestMapping(path = "/main", method = RequestMethod.GET, headers = "Magic-Word=please")
        public String headersMain(Model model) {
                model.addAttribute("flag", flags.getFlag("spring_mvc_6"));	// headers main
                return "flag";
        }
```

```
curl -H "Magic-Word: please"  'http://challenges.ctfd.io:30542/main'
<!DOCTYPE HTML>
<html>
<head>
    <title>Tenable CTF: Spring MVC</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
	<p >flag{flag6_ca1ddf}</p>
</body>
</html>
```
## Spring MVC 7

```
<body>
	<p th:text="'Hello, ' + ${name} + '!'" />
	<p th:if="${name == 'please'}">
	<span class="hidden" th:text="${@flagService.getFlag('hidden_flag')}" />
	</p>
	<p th:if="${#session.getAttribute('realName') == 'admin'}">
	<span th:text="${#session.getAttribute('sessionFlag')}" />
	</p>
</body>
```
```
curl http://challenges.ctfd.io:30542/?name=please
```

## One byte at the time

We know that it's a four byte IP address so first iterate all possible randoms to get the bytes. There seem to be 3 distinct octets so use them to iteratively plot options to the next character. Final code shown below.

```python
from pwn import *

candidates = [ord('f') ^ 0x64, ord('f') ^ 0x11, ord('f') ^ 0x76]

r = remote ('challenges.ctfd.io',30468)

r.recvuntil('Show me')

r.recvline() # flag

r.send('flag{f0ll0w_th3_whit3_r@bb1t}\n')

print(r.recvline())
print(r.recvline())
print(r.recvline())
print(r.recvline())
print(r.recvline())
x = r.recvline()
x = int(x,16)
print ("XOR :"+str(x))
for c in candidates:
    print (chr(c ^ x))
```

## Not JSON

First base64 decode the string and BSON deserialize. Then index using python:
```python
symbols = "abcdefghjiklmnopqrstuvwxyz_{}"
arr = [5,11,0,6,27,18,14,13,26,14,5,26,0,26,1,18,14,13,28]

for i in arr:
    print(symbols[i],end='')
print("")
```

## Hackerman

The flag is easily found from the .svg file using a text editor

## H4ck3R_m4n exp0sed! 1

Follow `tcp.stream eq 10` using Wireshark, extract to a .jpg file (raw) and view the file.


## H4ck3R_m4n exp0sed! 2 & 3 

extract ftp-data supersecure.7z file using filter `tcp.stream eq 6` using Wireshark. Follow the stream, save as raw and then decrypt the 7z file using the password from other file`tcp.stream eq 8` "The password for the compressed file is "bbqsauce"" 
 * Open pickle_nick.png to view flag 2
 * Use cyberchef to convert dataz file from hex and then from base 64, save the file and view flag 3
