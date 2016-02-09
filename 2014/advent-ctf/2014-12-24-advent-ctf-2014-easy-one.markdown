### Solved by bitvijays
In this reversing challenge, we are provided with the binary which when executed requests for the password.

>This is very easy crackme.

```
bitvijays@kali:~/Desktop/Advent$ ./easyone 
password: test
wrong
```

Running this executable in gdb-peda and diassembling the main function, we see that it has cmp command at main+195. Let's put a breakpoint at it and check the stack.

```
bitvijays@kali:~/Desktop/Advent$ gdb -q easyone
Reading symbols from /home/bitvijays/Desktop/Advent/easyone...(no debugging symbols found)...done.
gdb-peda$ pdisass main
Dump of assembler code for function main:
**Snip**
   0x000000000040069a <+173>:	mov    BYTE PTR [rbp-0x2f],0x44
   0x000000000040069e <+177>:	mov    BYTE PTR [rbp-0x13],0x0
   0x00000000004006a2 <+181>:	mov    rax,QWORD PTR [rbp-0x8]
   0x00000000004006a6 <+185>:	movzx  edx,BYTE PTR [rax]
   0x00000000004006a9 <+188>:	mov    rax,QWORD PTR [rbp-0x10]
   0x00000000004006ad <+192>:	movzx  eax,BYTE PTR [rax]
   0x00000000004006b0 <+195>:	cmp    dl,al
   0x00000000004006b2 <+197>:	je     0x4006c5 <main+216>
   0x00000000004006b4 <+199>:	mov    edi,0x4007a4
End of assembler dump.
**Snip**
gdb-peda$ br *main+195
Breakpoint 1 at 0x4006b0
gdb-peda$ run
password: hello
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe1e0 --> 0x6f6c6c6568 ('hello')
0008| 0x7fffffffe1e8 --> 0x1 
0016| 0x7fffffffe1f0 --> 0x1 
0024| 0x7fffffffe1f8 --> 0x40075d (<__libc_csu_init+77>:	add    rbx,0x1)
0032| 0x7fffffffe200 ("ADCTF_7H15_15_7oO_345y_FOR_M3")
0040| 0x7fffffffe208 ("15_15_7oO_345y_FOR_M3")
0048| 0x7fffffffe210 ("O_345y_FOR_M3")
0056| 0x7fffffffe218 --> 0x334d5f524f ('OR_M3')
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00000000004006b0 in main ()
```

***The Flag is ADCTF_7H15_15_7oO_345y_FOR_M3***
