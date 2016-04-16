### Solved by z0rex

rev2 was a reversing challenge worth 45 points.

> Little bit harder?  
> The answer will not be in the typical sctf{flag} format, so when you do get it,
> you must put it into the format by doing sctf{flag_you_found}

I opened `rev2` in gdb and disassembled the function main

```
# gdb rev2 -q
.
.
.
0x0000000000400665 <+15>:    mov    DWORD PTR [rbp-0x4],0x7d15d
0x000000000040066c <+22>:    mov    edi,0x4007a4
0x0000000000400671 <+27>:    call   0x400530 <puts@plt>
0x0000000000400676 <+32>:    lea    rax,[rbp-0x8]
0x000000000040067a <+36>:    mov    rsi,rax
0x000000000040067d <+39>:    mov    edi,0x4007c0
0x0000000000400682 <+44>:    mov    eax,0x0
0x0000000000400687 <+49>:    call   0x400550 <scanf@plt>
0x000000000040068c <+54>:    mov    eax,DWORD PTR [rbp-0x8]
0x000000000040068f <+57>:    cmp    eax,0x30dda83
0x0000000000400694 <+62>:    jne    0x400711 <main+187>
.
.
.
```

Looking at the assembly I quickly noticed that, just like rev1, eax is compared
against a hardcoded value.

```
0x000000000040068f <+57>:    cmp    eax,0x30dda83
```

This time it was `0x30dda83`. So let's see what this value translates to when
converted to an integer.

```.
gdb-peda$ p/d 0x30dda83
$1 = 51239555
```

Trying `51239555` as the password was successful

```
# ./rev2
What is the magic password?
51239555
Correct! Your flag is: 51196695
```

Flag: `sctf{51196695}`