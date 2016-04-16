### Solved by z0rex

rev1 was a reversing challenge worth 20 points.

> Let's start out with an easy, typical reversing problem.
> The answer will not be in the typical sctf{flag} format, so when you do get it,
> you must put it into the format by doing sctf{flag_you_found}

This was the easiest of the two reversing challenges `rev1` and `rev2`.

```
# gdb rev1 -q
.
.
.
0x0000000000400688 <+50>:    mov    edi,0x400760
0x000000000040068d <+55>:    mov    eax,0x0
0x0000000000400692 <+60>:    call   0x400550 <scanf@plt>
0x0000000000400697 <+65>:    mov    eax,DWORD PTR [rbp-0x4]
0x000000000040069a <+68>:    cmp    eax,0x5b74 # comparing our input against 0x5b74
0x000000000040069f <+73>:    jne    0x4006b7 <main+97>
0x00000000004006a1 <+75>:    lea    rax,[rbp-0x10]
0x00000000004006a5 <+79>:    mov    rsi,rax
0x00000000004006a8 <+82>:    mov    edi,0x400763
0x00000000004006ad <+87>:    mov    eax,0x0
0x00000000004006b2 <+92>:    call   0x400510 <printf@plt>
0x00000000004006b7 <+97>:    mov    eax,0x0
0x00000000004006bc <+102>:   leave  
0x00000000004006bd <+103>:   ret    
.
.
.
```

Now, on `main+68` we can see that eax, our input, is compared against the value 
`0x5b74`. So with some basic cli-fu I got the integer value, which I the tried
as the "magic password".

```
# echo $((0x5b74))
23412
```

Now, all that I have to do is to provide that as the password.

```
# ./rev1
What is the magic password?
23412
Correct! Your flag is: h4x0r!!!
```

Flag: sctf{h4x0r!!!}