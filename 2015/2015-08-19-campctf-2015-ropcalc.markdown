---
layout: post
title: "CampCTF 2015 ROPCalc"
date: 2015-08-19 20:59:16 -0400
author: [barrebas]
comments: true
categories: [campctf]
---

### Solved by barrebas

How could I resist a challenge called ropcalc? We're given the python server and a binary. We're supposed to write a ROP chain to satisfy certain conditions. The server.py will then pass random values for the registers to the binary, along with the ROP chain we provide. After execution of the ROP chain, it will check if the ROP chain has calculated the right answer. Pretty nifty, if you ask me!

The binary itself is actually packed with everything we need:

```
0000000000400b40 <sub_rax_rbx>:
  400b40:   sub    rax,rbx
  400b43:   ret    
  400b44:   data32 data32 nop WORD PTR cs:[rax+rax*1+0x0]

0000000000400b50 <imul_rax_rbx>:
  400b50:   imul   rax,rbx
  400b54:   ret    
  400b55:   data32 nop WORD PTR cs:[rax+rax*1+0x0]

0000000000400b60 <xchg_rax_rbx>:
  400b60:   xchg   rbx,rax
  400b62:   ret    
  400b63:   data32 data32 data32 nop WORD PTR cs:[rax+rax*1+0x0]
```

The expressions we need to satisfy look like this: `$rax + $rbx + 1337 (store result in rax)`

I'll just give the final exploit as it's not that difficult:

```python
#!/usr/bin/python
import struct, time
from socket import *

def q(x):
    return struct.pack("<Q", x)

def readtil(delim):
  buf = b''
  while not delim in buf:
      buf += s.recv(1)
  return buf

def toHex(x):
    payload = ""
    for i in x:
        payload += '%02x' % ord(i)
    return payload

  
def pwn():
    global s
    s = socket(AF_INET, SOCK_STREAM)
    s.connect(('challs.campctf.ccc.ac', 10109))
    
    readtil('line:')
    rop1 = ""
    rop1 += q(0x00400b30)
    rop1 += q(0x004046ab) # ret
    s.send(toHex(rop1)+'\n')
    
    print readtil('line:')
    
    rop2 = ""
    rop2 += q(0x400900) # pop_rcx
    rop2 += q(1337)
    rop2 += q(0x400b80) # add rax, rcx
    rop2 += q(0x400b30) # add rax, rbx
    rop2 += q(0x4046ab) # ret
    
    s.send(toHex(rop2)+'\n')
    
    print readtil('line:')
    
    rop3 = ""
    rop3 += q(0x400b50)
    rop3 += q(0x4046ab)
    
    s.send(toHex(rop3)+'\n')
    
    print readtil('line:')
    
    rop4 = ""
    rop4 += q(0x400900) # pop_rcx
    rop4 += q(31337)
    rop4 += q(0x400f90) 
    rop4 += q(0x400b50)
    rop4 += q(0x4046ab)
    
    s.send(toHex(rop4)+'\n')
    
    # $rcx + 23 * $rax + $rbx - 42 * ($rcx - 5 * $rdx - $rdi * $rsi) - $r8 + 2015
    rop5 = ""
    rop5 += q(0x400a20) # pop r10
    rop5 += q(23)
    rop5 += q(0x400d80) # imul rax, r10
    rop5 += q(0x400b80) # add rax, rcx
    rop5 += q(0x400b30) # add rax, rbx
    rop5 += q(0x400cd0) # sub rax, r8
    rop5 += q(0x400a20) # pop r10
    rop5 += q(2015)
    rop5 += q(0x400d60) # add rax, r10
    rop5 += q(0x4020e0) # imul rdi, rsi
    rop5 += q(0x4009c0) # pop r8
    rop5 += q(5)
    rop5 += q(0x401910) # imul rdx, r8
    rop5 += q(0x4018a0) # add rdx, rdi
    rop5 += q(0x401400) # sub rcx, rdx
    rop5 += q(0x4009c0) # pop r8
    rop5 += q(42)
    rop5 += q(0x401500) # imul rcx, r8
    rop5 += q(0x400b90) # sub rax, rcx
    rop5 += q(0x4046ab)
    
    s.send(toHex(rop5)+'\n')
    
    import telnetlib
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
pwn()
```

And in action:


```
bas@tritonal:~/bin/ccc/ropc$ python poc3.py 
 Correct!
--------------------------------------------------------------------------------
Level [2/5]
Create a ROP chain that calculates: $rax + $rbx + 1337 (store result in rax)
Enter your solution as a single hex encoded line:
 Correct!
--------------------------------------------------------------------------------
Level [3/5]
Create a ROP chain that calculates: $rax * $rbx (store result in rax)
Enter your solution as a single hex encoded line:
 Correct!
--------------------------------------------------------------------------------
Level [4/5]
Create a ROP chain that calculates: $rax * (31337 + $rbx) (store result in rax)
Enter your solution as a single hex encoded line:
 Correct!
--------------------------------------------------------------------------------
Level [5/5]
Create a ROP chain that calculates: $rcx + 23 * $rax + $rbx - 42 * ($rcx - 5 * $rdx - $rdi * $rsi) - $r8 + 2015 (store result in rax)
Enter your solution as a single hex encoded line: Correct!
--------------------------------------------------------------------------------
The flag is: CAMP15_c0342e0be22dc032de05aa637c8ee8a3

*** Connection closed by remote host ***
```

