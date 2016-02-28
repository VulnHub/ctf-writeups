I loaded up the binary in radare2 and examined available strings: 

```
[0x00400bb0]> iz
vaddr=0x00400fc4 paddr=0x00000fc4 ordinal=000 sz=7 len=6 section=.rodata type=ascii string=aseqcX
vaddr=0x00400fcb paddr=0x00000fcb ordinal=001 sz=11 len=10 section=.rodata type=ascii string=Take this:
vaddr=0x00400fd6 paddr=0x00000fd6 ordinal=002 sz=6 len=5 section=.rodata type=ascii string=Fail!
vaddr=0x00400fdc paddr=0x00000fdc ordinal=003 sz=6 len=5 section=.rodata type=ascii string=xfadd
```

Two interesting messages, "Take this" and "Fail" seemed to indicate that "Take this" would return the flag. I checked where it was being referenced:

```
[0x00400bb0]> axt 0x00400fcb
data 0x400d67 mov esi, str.Take_this: in sym.main
```

It was being called in main(), however, getting to this particular code branch meant that I needed to pass several checks:

![](/images/2016/pragyan/b_the_boss/01.png)

The first check examined to see if argc was less than or equal to 8:

```
|           0x00400cf4      c645b742       mov byte [rbp-local_49h], 0x42
|           0x00400cf8      837dac08       cmp dword [rbp-local_54h], 8 ; [0x8:4]=0
|       ,=< 0x00400cfc      0f8eca000000   jle 0x400dcc
|       |   0x00400d02      488b45a0       mov rax, qword [rbp-local_60h]
```

If it was, then it branches off to 0x400dcc:

```
|    |```-> 0x00400dcc      bed60f4000     mov esi, str.Fail_          ; "Fail!" @ 0x400fd6
|    |      0x00400dd1      bfc0206000     mov edi, obj._ZSt4cout__GLIBCXX_3.4 ; ".1-22ubuntu2) 5.2.1 20151010" @ 0x6020c0
|    |      0x00400dd6      e825fdffff     call sym.imp._ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
```

That was the fail message branch, so I definitely didn't want that. That meant I had to make sure argc was > 8, which meant the binary was expecting 8 arguments. Including the binary itself, that would set argc to 9. 

I ran the binary in gdb to see what it expected of each argument. Since the flag was inside the binary, I could just adjust the registers as needed to ensure that it took the correct code path. I started off with the arguments ```a a a a a a a a```. After stepping through the execution several times and adjusting the registers as needed, the winning argument turned out to be ```a a a a a a BB B```. Basically it was just interested in the last two arguments. 

```
# ./B_THE_B05S a a a a a a BB B
Take this:
xfaddBaseqcX
```

The flag was xfaddBaseqcX
