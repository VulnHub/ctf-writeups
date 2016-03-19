### Solved by superkojiman

This was a 250 point reversing challenge. The binary to reverse is 32-bit ELF called gibson. Attempting to strace it or run it within gdb results in an error:

```
gdb-peda$ r
Starting program: /root/work/gibson.orig
TRACING DETECTED!!!!! TRY HARDER!!!!
[Inferior 1 (process 838) exited with code 01]
```

To find out where this was happening, I loaded the binary into radare2:

```
# r2 -A gibson
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[*] Use -AA or aaaa to perform additional experimental analysis.
[x] Construct a function name for all fcn.* (.afna @@ fcn.*)
fcns: 24
xref: 82
sect: 802
code: 955
covr: 119 %
[0x080484b0]> afl
0x080483dc  35  3  sym._init
0x08048410  16  2  sym.imp.fgets
0x08048420  16  2  sym.imp.fclose
0x08048430  16  2  sym.imp.fwrite
0x08048440  16  2  sym.imp.puts
0x08048450  16  2  loc.imp.__gmon_start__
0x08048460  16  2  sym.imp.exit
0x08048470  16  2  sym.imp.__libc_start_main
0x08048480  16  2  sym.imp.fopen
0x08048490  16  2  sym.imp.fileno
0x080484a0  16  2  sym.imp.ptrace
0x080484b0  34  1  entry0
0x080484e0  4  1  sym.__x86.get_pc_thunk.bx
0x080484f0  43  4  sym.deregister_tm_clones
0x08048520  53  4  sym.register_tm_clones
0x08048560  30  3  sym.__do_global_dtors_aux
0x08048580  43  4  sym.frame_dummy
0x080485ab  137  5  sym.detect_gdb
0x08048634  51  4  sym.xor
0x08048667  83  10  sym.compare
0x080486ba  163  4  sym.main
0x08048760  97  4  sym.__libc_csu_init
0x080487d0  2  1  sym.__libc_csu_fini
0x080487d4  20  1  sym._fini
[0x080484b0]>
```

There's a function called sym.detect_gdb. Good place to start, so I examined it:

```
[0x080484b0]> s sym.detect_gdb
[0x080485ab]> pdf
╒ (fcn) sym.detect_gdb 137
│           ; var int local_ch     @ ebp-0xc
│           0x080485ab      55             push ebp
│           0x080485ac      89e5           mov ebp, esp
│           0x080485ae      83ec18         sub esp, 0x18
│           0x080485b1      83ec08         sub esp, 8
│           0x080485b4      68f0870408     push 0x80487f0
│           0x080485b9      68f2870408     push str._tmp               ; "/tmp" @ 0x80487f2
│           0x080485be      e8bdfeffff     call sym.imp.fopen
│           0x080485c3      83c410         add esp, 0x10
│           0x080485c6      8945f4         mov dword [ebp - local_ch], eax
│           0x080485c9      83ec0c         sub esp, 0xc
│           0x080485cc      ff75f4         push dword [ebp - local_ch]
│           0x080485cf      e8bcfeffff     call sym.imp.fileno
│           0x080485d4      83c410         add esp, 0x10
│             0x080485d7      83f805         cmp eax, 5
│       ┌─< 0x080485da      7e1a           jle 0x80485f6
│       │   0x080485dc      83ec0c         sub esp, 0xc
│       │   0x080485df      68f8870408     push str.GDB_DETECTED_______TERMINATING______ ; "GDB DETECTED!!!!!! TERMINATING!!!!!!" @ 0x80487f8
│       │   0x080485e4      e857feffff     call sym.imp.puts
│       │   0x080485e9      83c410         add esp, 0x10
│       │   0x080485ec      83ec0c         sub esp, 0xc
│       │   0x080485ef      6a01           push 1
│       │   0x080485f1      e86afeffff     call sym.imp.exit
│       │   ; JMP XREF from 0x080485da (sym.detect_gdb)
│       └─> 0x080485f6      83ec0c         sub esp, 0xc
│           0x080485f9      ff75f4         push dword [ebp - local_ch]
│           0x080485fc      e81ffeffff     call sym.imp.fclose
│           0x08048601      83c410         add esp, 0x10
│           0x08048604      6a00           push 0
│           0x08048606      6a01           push 1
│           0x08048608      6a00           push 0
│           0x0804860a      6a00           push 0
│           0x0804860c      e88ffeffff     call sym.imp.ptrace
│           0x08048611      83c410         add esp, 0x10
│           0x08048614      85c0           test eax, eax
│       ┌─< 0x08048616      791a           jns 0x8048632
│       │   0x08048618      83ec0c         sub esp, 0xc
│       │   0x0804861b      6820880408     push str.TRACING_DETECTED______TRY_HARDER____ ; "TRACING DETECTED!!!!! TRY HARDER!!!!" @ 0x8048820
│       │   0x08048620      e81bfeffff     call sym.imp.puts
│       │   0x08048625      83c410         add esp, 0x10
│       │   0x08048628      83ec0c         sub esp, 0xc
│       │   0x0804862b      6a01           push 1
│       │   0x0804862d      e82efeffff     call sym.imp.exit
│       │   ; JMP XREF from 0x08048616 (sym.detect_gdb)
│       └─> 0x08048632      c9             leave
╘           0x08048633      c3             ret
```

Right there I could see the error strings. Since everything was so neatly encapsulated into this function, I decided that the best way to bypass it was to just patch the binary and have it return as soon as it entered the function. 

```
[0x080485ab]> oo+
File gibson reopened in read-write mode
[0x080485ab]> wx c390
[0x080485ab]> oo
File gibson reopened in read-only mode
[0x080485ab]> pd 10
╒ (fcn) sym.detect_gdb 137
│           ; var int local_ch     @ ebp-0xc
│           0x080485ab      c3             ret
│           0x080485ac      90             nop
│           0x080485ad      e583           in eax, 0x83
│           0x080485af      ec             in al, dx
│           0x080485b0      1883ec0868f0   sbb byte [ebx - 0xf97f714], al
│           0x080485b6      870408         xchg dword [eax + ecx], eax
│           0x080485b9      68f2870408     push str._tmp               ; "/tmp" @ 0x80487f2
│           0x080485be      e8bdfeffff     call sym.imp.fopen
│           0x080485c3      83c410         add esp, 0x10
│           0x080485c6      8945f4         mov dword [ebp - local_ch], eax
```

Now I could run it within gdb:

```
# gdb -q gibson 
Reading symbols from gibson...(no debugging symbols found)...done. 
gdb-peda$ r 
Starting program: /root/work/gibson 
Please enter the correct password:bleh 
Nope. Try it the same but better. 
[Inferior 1 (process 1116) exited normally] 
gdb-peda$
```

But gdb isn't really needed to solve this. Looking at main() in radare2, I saw that it passed the password to a function called xor(), and then a function called compare() would check if the encoded password was the same as a hardcoded password in the binary. 

```
│           0x08048700      e82fffffff     call sym.xor
│           0x08048705      83c410         add esp, 0x10
│           0x08048708      83ec08         sub esp, 8
│           0x0804870b      68649b0408     push str._v_v__.M           ; obj.passwd
│           0x08048710      8d45f0         lea eax, [ebp - local_10h]
│           0x08048713      50             push eax
│           0x08048714      e84effffff     call sym.compare
```

The hardcoded encoded password is in `str._v_v__.M`. Dumping the hex shows:

```
[0x08048aab]> pxw 8@str._v_v__.M
0x08049b64  0x282b0b0b 0x00004d2e                      ..+(.M..
```

This is the encoded password, and not the flag. To get the flag, I needed to decode it, which meant reversing the xor() function. 

```
[0x080486ba]> s sym.xor
[0x08048634]> pdf
╒ (fcn) sym.xor 51
│           ; arg int arg_5h       @ ebp+0x5
│           ; arg int arg_8h       @ ebp+0x8
│           ; var int local_4h     @ ebp-0x4
│           ; CALL XREF from 0x08048700 (sym.xor)
│           0x08048634      55             push ebp
│           0x08048635      89e5           mov ebp, esp
│           0x08048637      83ec10         sub esp, 0x10
│           0x0804863a      c745fc000000.  mov dword [ebp - local_4h], 0
│       ┌─< 0x08048641      eb1c           jmp 0x804865f
│       │   ; JMP XREF from 0x08048663 (sym.xor)
│      ┌──> 0x08048643      8b55fc         mov edx, dword [ebp - local_4h]
│      ││   0x08048646      8b4508         mov eax, dword [ebp+arg_8h] ; [0x8:4]=0
│      ││   0x08048649      01d0           add eax, edx
│      ││   0x0804864b      8b4dfc         mov ecx, dword [ebp - local_4h]
│      ││   0x0804864e      8b5508         mov edx, dword [ebp+arg_8h] ; [0x8:4]=0
│      ││   0x08048651      01ca           add edx, ecx
│      ││   0x08048653      0fb612         movzx edx, byte [edx]
│      ││   0x08048656      83f26c         xor edx, 0x6c
│      ││   0x08048659      8810           mov byte [eax], dl
│      ││   0x0804865b      8345fc01       add dword [ebp - local_4h], 1
│      ││   ; JMP XREF from 0x08048641 (sym.xor)
│      │└─> 0x0804865f      837dfc05       cmp dword [ebp - local_4h], 5 ; [0x5:4]=257
│      └──< 0x08048663      7ede           jle 0x8048643
│           0x08048665      c9             leave
╘           0x08048666      c3             ret
```

It's a loop that iterates 6 times, and XOR's each character with 0x6c. This means the password is only 6 characters, which was obvious from the hexdump of `str._v_v__.M`. To get the actual password, I just XOR'd each character in the encoded password with 0x6c:

```python
>>> l = [0x0b, 0x0b, 0x2b, 0x28, 0x2e, 0x4d]
>>> for i in l: print chr(i ^ 0x6c),
...
g g G D B !
>>>
```

The flag is `ggGDB!`
