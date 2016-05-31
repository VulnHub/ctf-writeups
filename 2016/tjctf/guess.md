### Solved by superkojiman

This was an interesting challenge. Upon connecting to the service, we're expected to correctly guess 100 numbers in order to receive the flag. The source code to the binary is provided:

```
#include <stdio.h>
#include <stdlib.h>

int main() {
    int i;
    int num, guess;
    printf("Guess my number 100 times in a row and I'll give you the flag!\n");
    fflush(stdout);
    for (i = 0; i < 100; i++) {
        guess = 0;
        srand(time(NULL));
        num = rand();
        scanf("%d",&guess);
        if (guess == num) {
            printf("Correct!\n");
            fflush(stdout);
        } else {
            printf("Wrong.\n");
            printf("The answer was %d.\n",num);
            exit(0);
        }
    }
    printf("Congrats!\nFlag: ");
    char c;
    FILE *f = fopen("flag", "r");
    while ((c = fgetc(f)) != EOF) {
        putchar(c);
    }
    fclose(f);
    fflush(stdout);
}
```

The vulnerability is that srand(time(NULL)) gets called multiple times, instead of just once. Since it's called multiple times in the for loop, the same seed is used multiple times which makes it easy to guess the number. Here's a C file to demonstrate:

```
#include <stdio.h>
#include <stdlib.h>

int main() {
    int i = 0;
    for (i = 0; i < 20; i++) {
        srand(time(NULL));
        printf("%d\n", rand());
    }
    return 0;
}
```

And the output:

```
# ./demo
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
1091898205
```

Great, so the same number would be used multiple times each second. But how to guess the first number the server generated? To do that, I wrote my own helper C progam that would use the same seed as the server:

```
/* my-guess.c */
#include <stdio.h>
#include <stdlib.h>

int main() {
    srand(time(NULL));
    printf("%d", rand());
    return 0;
}
```

Next, I wrote a python script to call my helper binary, get the output, and send it to the server:

```
#!/usr/bin/env python

from pwn import *
from subprocess import *

r = remote("p.tjctf.org", 8007)
print r.recv()
for i in range(0, 100):
    guess = check_output(["./my-guess"])
    r.sendline(guess)
print r.recvall()
```

Run it against the server, and we get the flag:

```
# ./sploit.py
[+] Opening connection to p.tjctf.org on port 8007: Done
Guess my number 100 times in a row and I'll give you the flag!

[+] Recieving all data: Done (945B)
[*] Closed connection to p.tjctf.org port 8007
Correct!
Correct!
...
Correct!
Correct!
Congrats!
Flag: tjctf{n0t_so_r4ndom_4nymor3}
```


