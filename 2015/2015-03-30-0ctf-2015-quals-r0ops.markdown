### Solved by barrebas

`r0ops` was a reversing challenge, worth only 150 points. Based on the amount of points, I expected it to be relatively easy, but I was in for a ride down the rabbit hole...

The binary opens binds to a port and waits for incoming connections. Upon connecting with `nc`, nothing much happens. While trying to run it in `gdb`, I encountered the first anti-debugger trick:

```
;;; nice anti-debugger code
 dead448:   ba 00 00 00 00          mov    edx,0x0
 dead44d:   be 01 00 00 00          mov    esi,0x1
 dead452:   bf 02 00 00 00          mov    edi,0x2
 dead457:   e8 c4 32 55 f2          call   400720 <socket@plt>
 dead45c:   89 45 fc                mov    DWORD PTR [rbp-0x4],eax
 dead45f:   83 7d fc 03             cmp    DWORD PTR [rbp-0x4],0x3      ; anti-gdb trick
 dead463:   74 07                   je     dead46c <div+0x320>
 dead465:   b8 00 00 00 00          mov    eax,0x0
 dead46a:   eb 50                   jmp    dead4bc <div+0x370>
```

`gdb` opens more file descriptors. The binary rightly expects the socket handle to be file descriptor 3; if it encounter anything else, it must be because `gdb` is running.

Examing the output of `objdump`, I quickly learned that the main program is just a stub to load a ROP chain:

```
 ;;; accept calls
 dead3af:   eb 02                   jmp    dead3b3 <div+0x267>
 dead3b1:   52                      push   rdx
 dead3b2:   f2 48 83 ec 10          repnz sub rsp,0x10
 dead3b7:   ba 00 00 00 00          mov    edx,0x0
 dead3bc:   be 00 00 00 00          mov    esi,0x0
 dead3c1:   bf 03 00 00 00          mov    edi,0x3
 dead3c6:   e8 45 33 55 f2          call   400710 <accept@plt>
 dead3cb:   89 45 fc                mov    DWORD PTR [rbp-0x4],eax
 dead3ce:   8b 45 fc                mov    eax,DWORD PTR [rbp-0x4]
 dead3d1:   b9 00 00 00 00          mov    ecx,0x0
 dead3d6:   ba 00 10 00 00          mov    edx,0x1000
 dead3db:   be c0 10 0b 0e          mov    esi,0xe0b10c0
 dead3e0:   89 c7                   mov    edi,eax
 dead3e2:   e8 89 32 55 f2          call   400670 <recv@plt>
 dead3e7:   8b 45 fc                mov    eax,DWORD PTR [rbp-0x4]
 dead3ea:   89 c7                   mov    edi,eax
 dead3ec:   b8 00 00 00 00          mov    eax,0x0
 dead3f1:   e8 ca 32 55 f2          call   4006c0 <close@plt>
 dead3f6:   ba a0 f0 0a 0e          mov    edx,0xe0af0a0
 dead3fb:   be a0 00 0b 0e          mov    esi,0xe0b00a0
 dead400:   b8 00 02 00 00          mov    eax,0x200
 dead405:   48 89 d7                mov    rdi,rdx
 dead408:   48 89 c1                mov    rcx,rax
 dead40b:   f3 48 a5                rep movs QWORD PTR es:[rdi],QWORD PTR ds:[rsi]
 dead40e:   b8 c0 10 0b 0e          mov    eax,0xe0b10c0
 dead413:   48 89 c7                mov    rdi,rax                  ; input from socket
 dead416:   b8 c0 20 0b 0e          mov    eax,0xe0b20c0            ; storage
 dead41b:   48 89 c6                mov    rsi,rax
 dead41e:   b8 a0 f8 0a 0e          mov    eax,0xe0af8a0            ; contains a lot of addresses...
 dead423:   48 89 c4                mov    rsp,rax                  ; it's a ROP chain!
 dead426:   c3                      ret                             ; this generates a new program & jumps to it
 dead427:   c9                      leave  
 dead428:   c3                      ret                             
```

I set a breakpoint at the `ret` instruction, just before the ROP chain kicks off. I then dumped and copied the ROP chain and used some shell-fu to clean up the output:

```bash
$ head stackdump 
0xe0af8a0:  0x0dead1f4  0x00000000  0x00000008  0x00000000
0xe0af8b0:  0x0dead271  0x00000000  0xbeef0095  0x1337dead
0xe0af8c0:  0x0dead123  0x00000000  0x0dead0ed  0x00000000
0xe0af8d0:  0x0dead204  0x00000000  0x0dead267  0x00000000
0xe0af8e0:  0x0dead0f8  0x00000000  0x0dead103  0x00000000
0xe0af8f0:  0x0dead0ed  0x00000000  0x0dead27a  0x00000000
0xe0af900:  0x0dead20e  0x00000000  0x0dead0f8  0x00000000
0xe0af910:  0x0dead1ec  0x00000000  0x0000cafe  0x00000000
0xe0af920:  0x0dead141  0x00000000  0x0dead0ed  0x00000000
0xe0af930:  0x0dead204  0x00000000  0x0dead284  0x00000000
$ cat stackdump | sed 's/0x//g' | awk '{print $3 $2"\n" $5 $4}' > ropchain
```

Now, the ropchain contained only bare addresses. This is were the second obfuscation step came into place: each ROP gadget starts with a `jmp` which jumps in the middle of another instruction. Because of this, the disassembly cannot be trusted. Instead, I manually looked up all the ROP gadgets and pieced them together. The ROP chain is quite ingenious, although it also contains tons of redundant instruction:

```
000000000dead1f4    pop rcx
0000000000000008
000000000dead271    pop r9                          ; r9 = 1337deadbeef0095
1337deadbeef0095
000000000dead123    mov    rax,QWORD PTR [rdi]      ; [rdi+0] first QWORD of input
000000000dead0ed    add    rsi,0x8                  ; rsi=8
000000000dead204    mov    QWORD PTR [rsi],rax      ; rsi=8
000000000dead267    mov    r8,QWORD PTR [rsi]       ; rsi=8 r8 = first QWORD of input
000000000dead0f8    sub    rsi,0x8                  ; rsi=0
000000000dead103    add    rdi,0x8                  ; rdi=8
000000000dead0ed    add    rsi,0x8                  ; rsi=8 (no-op)
000000000dead27a    mov    QWORD PTR [rsi],r9       ; rsi=8 [rsi] = 1337deadbeef0095
000000000dead20e    mov    rax,QWORD PTR [rsi]      ; rsi=8 rax = 1337deadbeef0095
000000000dead0f8    sub    rsi,0x8                  ; rsi=0
000000000dead1ec    pop    rbx                      ; rbx = cafe
000000000000cafe
000000000dead141    imul   rax,rbx                  ; rax ==> 0x2724090c079825d6
000000000dead0ed    add    rsi,0x8                  ; rsi=8
000000000dead204    mov    QWORD PTR [rsi],rax      ; rax = 0x1337deadbeef0095*0xcafe
...continues...
```

It basically takes the first QWORD of the input, sent over the socket, and then proceeds to generate a special constant. This is used later to compare against. I followed the rest of the ROP chain, and it basically does the following: it repeatedly multiplies the QWORD of our input with itself. At set intervals, it will multiply this value with the original QWORD. After a fixed number of iterations, it compares the resulting QWORD to the generated magic constant. The ROP chain uses a clever mechanism to implement conditional looping:


```
000000000dead1ec    pop    rbx                      ; rbx=0
0000000000000000
000000000dead1fc    pop    rdx                      ; rdx=1d8; adjustment for rsp!
00000000000001d8                                    ; 
000000000dead19b    0xdead19f:  cmp    rax,rbx      ; rax contains a counter used to iterate; 
                    0xdead1a2:  jne    0xdead1a7    ; -> ret; if rax != rbx, continue
                    0xdead1a4:  add    rsp,rdx      ; when it reaches zero, control is passed to the next gadget, located at rsp+0x1d8 
```

Clever stuff, but horrible to trace. There were a lot of jumps and no ops to throw me off. For instance, a gadget would `add rsi, 8` and the next one would `sub rsi, 8`, effectively doing nothing (except annoying me and wearing out my Enter key).

Breaking the chain
------------------

The ROP chain repeats this process eight times, so we need to send eight QWORDS over the socket. For each QWORD, a new magic constant is generated (taking the former value, multiplying by `0xcafe` and adding `0xbeef`). To inspect what was going on, I set breakpoints on two very important ROP gadgets:

```
Breakpoint 1, 0x000000000dead145 in ?? ()
1: x/i $rip
=> 0xdead145:   imul   rax,rbx


Breakpoint 2, 0x000000000dead1ae in ?? ()
1: x/i $rip
=> 0xdead1ae:   cmp    rax,rbx
```

This allowed me to dump each value that was generated, and finally see which values are being compared by the binary (one of which was the magic constant). 

I briefly considered bruteforcing the entire 64-bit range, but this was *way* too slow, even in C. I focussed on creating a function that emulates what is done with the first QWORD. After squashing a bug, I ended up with the following python code:

```python
def p(x, n):
    while n:
        x = (x*x) & 0xffffffffffffffff
        n -= 1
    return x
    
def c(i):
    x = (p(i, 3) * i) & 0xffffffffffffffff
    x = (p(i, 4) * x) & 0xffffffffffffffff
    x = (p(i, 10) * x) & 0xffffffffffffffff
    x = (p(i, 12) * x) & 0xffffffffffffffff
    return (p(i, 13) * x) & 0xffffffffffffffff

print hex(c(0x4242424241414141)) # remember, little endian ;)
```

Then I noticed something crucial. As I entered variations of `0x4242424241414141`, the last byte of the generated value was only dependent on the last byte of the input (by chance it was also `0x41`)! This gave me an idea...

Byte-by-byte
------------

I found I could bruteforce the correct value for each QWORD, going one byte at a time! After a while (and squashing the aforementioned bug by careful tracing of the ROP chain) I came up with the following python code:

```python
import struct

def q(x):
    return struct.pack('<Q', x)
    
def p(x, n):
    while n:
        x = (x*x) & 0xffffffffffffffff
        n -= 1
    return x
    
def c(i):
    x = (p(i, 3) * i) & 0xffffffffffffffff
    x = (p(i, 4) * x) & 0xffffffffffffffff
    x = (p(i, 10) * x) & 0xffffffffffffffff
    x = (p(i, 12) * x) & 0xffffffffffffffff
    return (p(i, 13) * x) & 0xffffffffffffffff


key_list = []
check = 0x1337deadbeef0095
for u in range(8):
    check = ((check * 0xcafe) + 0xbeef) & 0xffffffffffffffff

    key = 0
    for i in range(8):
        for z in xrange(1,0xff):
            # ugly, but works: it basically only compares the output of the c() function
            # up to the byte it's bruteforcing
            if (c(key | (z << (i*8))) & (0xff << i*8)) == (check & (0xff << i*8)):
                key += (z << (i * 8))
                break

    print "[+] key {}: {} -> {} == {}".format(u, hex(key), hex(c(key)), hex(check))
    key_list.append(key)
    
# send all the generated keys as little-endian QWORDS to the binary
from socket import *
global s
s=socket(AF_INET, SOCK_STREAM)
s.connect(('localhost', 13337))

payload = ''
for key in key_list:
    payload += q(key)
    
s.send(payload+'\n')
print s.recv(1000)
    
s.close()
```

The ROP chain went through its hoops and landed here, dumping the flag!

```
000000000dead1aa    0xdead1ae:  cmp    rax,rbx
                    0xdead1b1:  je     0xdead1b6  ; if rax == rbx, the special constant and the value generated from our QWORD match
                    0xdead1b3:  add    rsp,rdx    ; if rax == rbx, this is skipped...
                        0xdead1b6:  ret               ;
000000000dead1fc    pop    rdx                    ; ...and the ROP chain continues here...
fffffffffffffc38
000000000dead1d7    loop   0xdead1db              ; ...if all eight QWORDS check out... (rcx contained 8 at the start)
000000000dead33c
   0xdead340:   sub    rsp,0x10                   ; ...then control is passed here
   0xdead344:   mov    edi,0xdead544
   0xdead349:   call   0x400680 <puts@plt>
   0xdead34e:   mov    edi,0xdead54e
   0xdead353:   mov    eax,0x0
   0xdead358:   call   0x4006a0 <printf@plt>
   0xdead35d:   mov    DWORD PTR [rbp-0x4],0x0
   0xdead364:   jmp    0xdead38b
   0xdead366:   mov    eax,DWORD PTR [rbp-0x4]
   0xdead369:   cdqe   
   0xdead36b:   mov    rax,QWORD PTR [rax*8+0xe0b10c0]
   0xdead373:   mov    eax,eax
   0xdead375:   mov    rsi,rax
   0xdead378:   mov    edi,0xdead55d
   0xdead37d:   mov    eax,0x0
   0xdead382:   call   0x4006a0 <printf@plt>
   0xdead387:   add    DWORD PTR [rbp-0x4],0x1
   0xdead38b:   cmp    DWORD PTR [rbp-0x4],0x7
   0xdead38f:   jle    0xdead366
   0xdead391:   mov    edi,0xdead564
   0xdead396:   call   0x400680 <puts@plt>            ; dumps flag in console!
   0xdead39b:   mov    eax,0x0
   0xdead3a0:   call   0xdead3af
   0xdead3a5:   leave  
   0xdead3a6:   ret
```

The output of the script and binary:

```bash
bas@tritonal:~/tmp/0ctf/r0ops$ ./r0ops & python ./bf.py
[1] 4471
[+] key 0: 0xd5b028b6c97155a5L -> 0x2724090c0798e4c5L == 0x2724090c0798e4c5L
[+] key 1: 0x51a2c3e8e288fa45 -> 0x44e477ee2e372c65L == 0x44e477ee2e372c65L
[+] key 2: 0x561720a3f926b105 -> 0xa150eec963c67d25L == 0xa150eec963c67d25L
[+] key 3: 0xa325ec548e4e0385L -> 0xeab7d48b9db01ba5L == 0xeab7d48b9db01ba5L
[+] key 4: 0x5369761ad6ccde85 -> 0xf01b0cf36a8c5ea5L == 0xf01b0cf36a8c5ea5L
[+] key 5: 0x9475802813002885L -> 0x930eeb9679f4d8a5L == 0x930eeb9679f4d8a5L
[+] key 6: 0xcadd6a0bdc679485L -> 0xaeb27b8833e1e4a5L == 0xaeb27b8833e1e4a5L
[+] key 7: 0x7d67b37124bcbc85 -> 0x2a900a13b88bcca5L == 0x2a900a13b88bcca5L

YOU WIN!

FLAG IS: 0ctf{c97155a5e288fa45f926b1058e4e0385d6ccde8513002885dc67948524bcbc85}
```

Good stuff! Funny to see a ROP chain "from the other side" :)

