### Solved by superkojiman

This was a 100 point binary exploitation challenge. Much like the other binary challenges, the flag is somewhere in the binary. Using radare2, I looked at what strings were available: 

```
[0x00400dd0]> iz
vaddr=0x00401818 paddr=0x00001818 ordinal=000 sz=12 len=11 section=.rodata type=ascii string=%Y-%m-%d.%X
vaddr=0x00401824 paddr=0x00001824 ordinal=001 sz=5 len=4 section=.rodata type=ascii string=AHU8
vaddr=0x00401829 paddr=0x00001829 ordinal=002 sz=11 len=10 section=.rodata type=ascii string=Take it :
vaddr=0x00401837 paddr=0x00001837 ordinal=003 sz=14 len=6 section=.rodata type=wide string=81潄瑮朠癩
vaddr=0x00401855 paddr=0x00001855 ordinal=004 sz=11 len=10 section=.rodata type=ascii string=SUE8FdaE13
vaddr=0x00401860 paddr=0x00001860 ordinal=005 sz=6 len=5 section=.rodata type=ascii string=MA5T3
vaddr=0x00401866 paddr=0x00001866 ordinal=006 sz=8 len=7 section=.rodata type=ascii string=DS8UE81
vaddr=0x0040186e paddr=0x0000186e ordinal=007 sz=9 len=8 section=.rodata type=ascii string=SDS1B333
vaddr=0x00401877 paddr=0x00001877 ordinal=008 sz=6 len=5 section=.rodata type=ascii string=SU3W0
```

The "Take it" string caught my eye as the previous challenge I solved, B THE B05S, had a similar message that preceeded the printing of the flag. Where was it being referenced from?

```
[0x00400dd0]> axt 0x00401829
data 0x4011ac mov esi, str.Take_it_: in sym.main
```

Looks like it was being used in main(), so I checked out what main() looked like:

```
|       |   0x0040118f      e84cfbffff     call sym.imp.atoi
|       |   0x00401194      89857cffffff   mov dword [rbp-local_84h], eax
|       |   0x0040119a      8b8578ffffff   mov eax, dword [rbp-local_88h]
|       |   0x004011a0      3b857cffffff   cmp eax, dword [rbp-local_84h]
|      ,==< 0x004011a6      0f8598000000   jne 0x401244
|      ||   0x004011ac      be29184000     mov esi, str.Take_it_:      ; "Take it : " @ 0x401829
|      ||   0x004011b1      bfe0206000     mov edi, obj.__TMC_END__    ; "GCC: (Ubuntu 5.2.1-22ubuntu2) 5.2.1 20151010" @ 0x6020e0
|      ||   0x004011b6      e8f5faffff     call sym.imp._ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
```

So to get to the winning branch, I needed to make sure that ```cmp eax, dword [rbp-local_84h]``` was equal. The first branch in main occurs at 0x0041178

```
| 0x00401161 898578ffffff   mov dword [rbp-local_88h], eax    |
| 0x00401167 488b8550ffff.  mov rax, qword [rbp-local_b0h]    |
| 0x0040116e 4883c008       add rax, 8                        |
| 0x00401172 488b00         mov rax, qword [rax]              |
| 0x00401175 4885c0         test rax, rax                     |
| 0x00401178 0f84e4000000   je 0x401262 ;[j]                  |
```

This checked to see if an argument was provided, and if one wasn't, the it printed out "Don't give up" and exited. If an argument was provided, it would run atoi() on it:

```
|       ,=< 0x00401178      0f84e4000000   je 0x401262
|       |   0x0040117e      488b8550ffff.  mov rax, qword [rbp-local_b0h]
|       |   0x00401185      4883c008       add rax, 8
|       |   0x00401189      488b00         mov rax, qword [rax]
|       |   0x0040118c      4889c7         mov rdi, rax
|       |   0x0040118f      e84cfbffff     call sym.imp.atoi
|       |   0x00401194      89857cffffff   mov dword [rbp-local_84h], eax
|       |   0x0040119a      8b8578ffffff   mov eax, dword [rbp-local_88h]
|       |   0x004011a0      3b857cffffff   cmp eax, dword [rbp-local_84h]
|      ,==< 0x004011a6      0f8598000000   jne 0x401244
```

So I knew I had to provide a number. I picked 123, and set a breakpoint at 0x004011a0, which is the compare instruction right before execution branches off to "Don't give up" or "Take it". Once I hit the breakpoint, I examined the values in eax and rbp-0x84: 

```
pwndbg> p $eax
$4 = 0xb06
pwndbg> x/wx $rbp-0x84
0x7ffd858b224c: 0x0000007b
pwndbg> p/d 0x7b
$5 = 123
```

So it was comparing the argument I provided with a value 0xb06. I noticed that the value in eax was changing after each run. While it was possible to brute force the value it was expecting, I went with just changing the value in rbp-0x84 to whatever was in eax. I was in a debugger after all. 

```
pwndbg> p $rbp-0x84
$8 = (void *) 0x7ffed4f9f94c
pwndbg> set *0x7ffed4f9f94c=$eax
pwndbg> x/wx $rbp-0x84
0x7ffed4f9f94c: 0x00000b06
```

Sure enough, it branched off into the "Take it" branch. 

```
    0x4011a0 <main+432>    cmp    eax, dword ptr [rbp - 0x84]
    0x4011a6 <main+438>    jne    0x401244 <main+596>

 => 0x4011ac <main+444>    mov    esi, 0x401829
    0x4011b1 <main+449>    mov    edi, 0x6020e0                             # 0x6020e0 <std::cout@@GLIBCXX_3.4>
    0x4011b6 <main+454>    call   0x400cb0
```

Was that sufficient? I let the binary continue to see if it would spit out an error message or give me the flag. 

```
pwndbg> c
Continuing.
Take it : N080DYK1ll3D8AHU8AL1
[Inferior 1 (process 41976) exited normally]
```

It gave me the flag! 100 points in the bag!
