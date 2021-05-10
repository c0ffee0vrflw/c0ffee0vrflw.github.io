---
title: Vulnhub "brainpan" walkthrough
published: true
---

For everyone who's trying to get some hands-on practice on buffer overflows, without getting too much of a headache,
brainpan is a great box to begin with, as it gives the plain basics of how you should approach buffer overflows.
I suggest trying to solve the box on your own before moving on with the walkthorugh, but if you must use it... try not to read the whole thing,
but a bit just to get yourself un-stuck.

First of all, after loading up the brainpan machine, which we'll ignore is an ubuntu machine just for the means of practicing,
we'll run up **fping** to scan our network (VMWare, all machines are bridged).

```
fping -a -g -m -q 111.61.0.0/24

111.61.0.7
111.61.0.117
111.61.0.153
```

Knowing that .153 is my kali machine, and that .7 is my windows machine, I now know that 111.61.0.117 is the brainpan machine.
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
We'll go and browse to the site under port 10000 and we'll get:

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

As we can see theres a directory called "bin", when we go to that directory we get the directory listing for /bin/, which includes a file named "brainpan.exe".
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

Loading up brainpan in **Immunity Debugger**, 
