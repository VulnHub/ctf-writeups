rock
---

A reversing challenge! We're given a 64-bit ELF executable. When we enter 'AAAA', the program says

```
bas@debian:~/csaw/rock$ ./rock
AAAA
-------------------------------------------
Quote from people's champ
-------------------------------------------
*My goal was never to be the loudest or the craziest. It was to be the most entertaining.
*Wrestling was like stand-up comedy for me.
*I like to use the hard times in the past to motivate me today.
-------------------------------------------
Checking....
Too short or too long
```

I first fired up gdb and started the program. Halting the execution, I search for the string "Too short"

```
Temporary breakpoint 1, 0x0000000000401260 in ?? ()
gdb-peda$ find "Too short"
Searching for 'Too short' in: None ranges
Found 1 results, display max 1 items:
rock : 0x401a1f ("Too short or too long")
```

Now in the disassembly, I looked for references to that string

```
  4016d7:       48 83 f8 1e             cmp    rax,0x1e
  4016db:       0f 95 c0                setne  al
  4016de:       84 c0                   test   al,al
  4016e0:       74 26                   je     401708 <_Unwind_Resume@plt+0x808>
  4016e2:       be 1f 1a 40 00          mov    esi,0x401a1f
```

Slightly above the reference, there's a comparision between what is presumed to be a length and the number 0x1e (30 in decimal). This appears to be the length of the string the program expects. 

```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
-------------------------------------------
Quote from people's champ
-------------------------------------------
*My goal was never to be the loudest or the craziest. It was to be the most entertaining.
*Wrestling was like stand-up comedy for me.
*I like to use the hard times in the past to motivate me today.
-------------------------------------------
Checking....
You did not pass 0
```

Repeating the process for the string "You did not pass", I found

```
  40186b:       eb 36                   jmp    4018a3 <_Unwind_Resume@plt+0x9a3>
  40186d:       be 3b 1a 40 00          mov    esi,0x401a3b
```

The unconditional jmp means that the 'you did not pass' is reached from another piece of code:

```
  40182d:       e8 2e f6 ff ff          call   400e60 <_ZNSsixEm@plt>
  401832:       0f b6 00                movzx  eax,BYTE PTR [rax]
  401835:       38 c3                   cmp    bl,al
  401837:       0f 94 c0                sete   al
  40183a:       84 c0                   test   al,al
  40183c:       74 2f                   je     40186d <_Unwind_Resume@plt+0x96d>
```

OK, great. Firing up gdb again, placing a breakpoint at 0x401832 leaves me here:

```
RAX: 0x604138 ('>' <repeats 30 times>, "}")
RBX: 0x46 ('F')
...snip...
=> 0x401832:	movzx  eax,BYTE PTR [rax]
```

Hmm. The thirty As have been replaced with '>' and the first one is compared with 'F'. Smells like the beginning of 'FLAG' won't you say? I step over the 'je 0x40186d':

```
=> 0x40183c:	je     0x40186d
 | 0x40183e:	mov    esi,0x401a35
...snip...
gdb-peda$ set $rip=0x40183e
```

and end up here:

```
RAX: 0x604139 ('>' <repeats 29 times>, "}")
RBX: 0x4c ('L')
...
=> 0x401832:	movzx  eax,BYTE PTR [rax]
```

Repeating this process (actually, patching the JE to a JNE) yields the string the program wants:

```
FLAG23456912365453475897834567
```

But the input is mangled so this is not the flag. I noticed that each byte gets transformed to another byte, regardless of position or previous byte. All I had to do was to find the corresponding values (by running the program a few times, breaking at 0x401832 and observing the value pointed to by RAX). 

Finally, this gave the flag:

```python
>>> import maketrans from string
>>> intab = ">?@ABCDEFGH)*+,\r\016\017\020\021\022\023\024\025\026\027^_`abcdefghIJKL-./01234567mnopqrstuv89"
>>> outab = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789{\\"
>>> tab = maketrans(intab, outab)>>> "FLAG23456912365453475897834567".translate(tab)
'IoDJuvwxy\\tuvyxwxvwzx{\\z{vwxyz'
```

Entering 'IoDJuvwxy\tuvyxwxvwzx{\z{vwxyz':

```
bas@debian:~/csaw/rock$ ./rock
IoDJuvwxy\tuvyxwxvwzx{\z{vwxyz
-------------------------------------------
Quote from people's champ
-------------------------------------------
*My goal was never to be the loudest or the craziest. It was to be the most entertaining.
*Wrestling was like stand-up comedy for me.
*I like to use the hard times in the past to motivate me today.
-------------------------------------------
Checking....
Pass 0
Pass 1
Pass 2
Pass 3
Pass 4
Pass 5
Pass 6
Pass 7
Pass 8
Pass 9
Pass 10
Pass 11
Pass 12
Pass 13
Pass 14
Pass 15
Pass 16
Pass 17
Pass 18
Pass 19
Pass 20
Pass 21
Pass 22
Pass 23
Pass 24
Pass 25
Pass 26
Pass 27
Pass 28
Pass 29
/////////////////////////////////
Do not be angry. Happy Hacking :)
/////////////////////////////////
Flag{IoDJuvwxy\tuvyxwxvwzx{\z{vwxyz}
```


