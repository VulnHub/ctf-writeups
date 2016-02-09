---
layout: post
title: "DefCon 2015 Quals mathwhiz"
date: 2015-05-18 12:20:17 -0400
author: [superkojiman]
comments: true
categories: [defcon]
---

### Solved by superkojiman

This was a fairly easy problem. Connecting to the server will return a math problem, and we need to return the solution. The problems start off very easy, two numbers that need to be added or subtracted. After that you start getting three numbers. Then come the parentheses, brackets, and braces. Once you get past that, numbers are replaced with words (eg: 3 becomes THREE). Finally you need to deal with numbers with exponents. I hacked together the following script that solves those problems: 

```python
#!/usr/bin/env python
from pwn import *
import sys

r = remote("mathwhiz_c951d46fed68687ad93a84e702800b7a.quals.shallweplayaga.me", 21249)

nums = { "ONE":1, 
        "TWO":2,
        "THREE":3,
        "FOUR":4,
        "FIVE":5,
        "SIX":6,
        "SEVEN":7,
        "EIGHT":8,
        "NINE":9,
        "ZERO":0
}

while (True):
    p = r.recvline().split("=")[0]

    # we're done with the problems! get the flag
    if "You won" in p:
        print r.recv()
        break

    for k, v in nums.iteritems():
        if k in p:
            p = p.replace(str(k), str(nums[k]))

    # replace any brackets/braces with parentheses
    p = p.replace("[", "(")
    p = p.replace("]", ")")
    p = p.replace("{", "(")
    p = p.replace("}", ")")

    # eval doesn't like ^, so replace with **
    p = p.replace("^", "**")

    # solve it with eval()
    s = eval(p)

    print "problem %s , solution: %d" % (p,  s)
    r.send(str(s) + "\n")
```

Run the script and it will start receiving problems and we send in the solutions. Eventually we get:

```text
koji@pwnbox:$ ./mathsolve.py 
[+] Opening connection to mathwhiz_c951d46fed68687ad93a84e702800b7a.quals.shallweplayaga.me on port 21249: Done
problem 2 + 1  , solution: 3
problem 3 - 1  , solution: 2
problem 1 + 2  , solution: 3
problem 3 - 2  , solution: 1
problem 3 - 2  , solution: 1
.
.
.
problem 2 + (2 - 1)  , solution: 3
problem 3 - 3 - 2 + 3  , solution: 1
problem 1 + 1 + 1  , solution: 3
problem 2 - 3 + 2 + 1  , solution: 2
problem 1 + 1 + 1 - 1  , solution: 2
problem 3 + 2 - 3  , solution: 2
The flag is: Farva says you are a FickenChucker and you'd better watch Super Troopers 2
```

The flag is returned to us once we've solved all the challenges: **Farva says you are a FickenChucker and you'd better watch Super Troopers 2**
