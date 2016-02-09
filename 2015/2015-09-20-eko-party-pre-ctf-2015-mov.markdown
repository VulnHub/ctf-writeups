---
layout: post
title: "Eko Party Pre-CTF 2015 MOV"
date: 2015-09-20 23:47:55 -0400
author: [superkojiman]
comments: true
categories: [ekoparty]
---

32-bit binary. If we check out the disassembly we see that it's composed mostly of mov instructions (hence the name, MOV). Run it:

```
# ./MOV 
        __                              __          
  ____ |  | ______ ___________ ________/  |_ ___.__.
_/ __ \|  |/ /  _ \\____ \__  \\_  __ \   __<   |  |
\  ___/|    <  <_> )  |_> > __ \|  | \/|  |  \___  |
 \___  >__|_ \____/|   __(____  /__|   |__|  / ____|
     \/     \/     |__|       \/             \/     

Processing...
Error
```

I solved this quite by accident. Looking through the strings I followed an xref from:

```
.data:0804D420 a__             db '     \/     \/     |__|       \/             \/     ',0Ah
```

This xrefs from 

```
text:0804A116                 mov     eax, offset a__ ; "     \\/     \\/     |__|       \\/    "...
```

Looks like one of the lines in the ASCII banner, so I set a breakpoint there. Ran it in gdb:

```
# gdb -q MOV
Reading symbols from MOV...(no debugging symbols found)...done.
gdb-peda$ br *0x0804A116
Breakpoint 1 at 0x804a116
gdb-peda$ r
Starting program: /root/Desktop/MOV 
        __                              __          
  ____ |  | ______ ___________ ________/  |_ ___.__.
_/ __ \|  |/ /  _ \\____ \__  \\_  __ \   __<   |  |
\  ___/|    <  <_> )  |_> > __ \|  | \/|  |  \___  |
 \___  >__|_ \____/|   __(____  /__|   |__|  / ____|
[----------------------------------registers-----------------------------------]
EAX: 0x83f6650 --> 0x85f6664 ("All_You_Need_Is_m0v")
EBX: 0x8000 
ECX: 0x0 
EDX: 0x85f6664 ("All_You_Need_Is_m0v")
ESI: 0xbf88e42c --> 0xbf88f557 ("XDG_VTNR=7")
EDI: 0x804827c (mov    DWORD PTR ds:0x83f6660,esp)
EBP: 0x0 
ESP: 0x85f6660 --> 0x804d457 (" \\___  >__|_ \\____/|   __(____  /__|   |__|  / ____|\n")
EIP: 0x804a116 (mov    eax,0x804d420)
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a107:   mov    eax,DWORD PTR [edx*4+0x83f6690]
   0x804a10e:   mov    edx,DWORD PTR ds:0x81f6630
   0x804a114:   mov    DWORD PTR [eax],edx
=> 0x804a116:   mov    eax,0x804d420
   0x804a11b:   mov    ds:0x804d57c,eax
   0x804a120:   mov    eax,ds:0x804d57c
   0x804a125:   mov    eax,eax
   0x804a127:   mov    ds:0x81f6630,eax
[------------------------------------stack-------------------------------------]
0000| 0x85f6660 --> 0x804d457 (" \\___  >__|_ \\____/|   __(____  /__|   |__|  / ____|\n")
0004| 0x85f6664 ("All_You_Need_Is_m0v")
0008| 0x85f6668 ("You_Need_Is_m0v")
0012| 0x85f666c ("Need_Is_m0v")
0016| 0x85f6670 ("_Is_m0v")
0020| 0x85f6674 --> 0x76306d ('m0v')
0024| 0x85f6678 --> 0x0 
0028| 0x85f667c --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804a116 in ?? ()
```

The flag is in the stack :)  ***EKO{All_You_Need_Is_m0v}***

