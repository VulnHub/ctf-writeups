AlexCTF Scripting 1: Math bot
---

### Solved by rasta_mouse

For this challenge, we are provided the following clue:

```
It is well known that computers can do tedious math faster than human.
nc 195.154.53.62 1337
```

Connecting to the service, we get the following output:

```
➜  mathbot nc 195.154.53.62 1337 
                __________
         ______/ ________ \______
       _/      ____________      \_
     _/____________    ____________\_
    /  ___________ \  / ___________  \
   /  /XXXXXXXXXXX\ \/ /XXXXXXXXXXX\  \
  /  /############/    \############\  \
  |  \XXXXXXXXXXX/ _  _ \XXXXXXXXXXX/  |
__|\_____   ___   //  \\   ___   _____/|__
[_       \     \  X    X  /     /       _]
__|     \ \                    / /     |__
[____  \ \ \   ____________   / / /  ____]
     \  \ \ \/||.||.||.||.||\/ / /  /
      \_ \ \  ||.||.||.||.||  / / _/
        \ \   ||.||.||.||.||   / /
         \_   ||_||_||_||_||   _/
           \     ........     /
            \________________/

Our system system has detected human traffic from your IP!
Please prove you are a bot
Question  1 :
105324290008660117721586104991656 - 59133588453337148078157371850961 =
```

The goal of the challenge is to solve the maths puzzles.  After some trial and error, it turns out we have to solve 250 before receiving the flag.  This is my final script.

``` python
#!/usr/bin/env python

import sys
from pwn import *

## make connection
c = remote('195.154.53.62',1337)

## get the ascii bot and other junk
c.recvuntil(':')

sys.stdout.write('Running')

for n in range(250):

    sys.stdout.write('.')

    ## get the first question
    question = c.recvuntil('=').strip()

    # split the parts
    q = question.split()
    first = int(q[0])
    operator = str(q[1])
    second = int(q[2])

    ## check operator
    if operator == "+":
        answer = first + second
    elif operator == "-":
        answer = first - second
    elif operator == "*":
        answer = first * second
    elif operator == "/":
        answer = first / second
    elif operator == "%":
        answer = first % second
    else:
        print "error"

    # print answer

    ## send the answer
    c.sendline(str(answer))

    ## get and discard prompt for next question
    ## e.g "Question 2:"
    c.recvuntil(':')

## print until the end of the flag, i.e. final brace

print "\nFlag is: " + c.recvuntil('}')

c.close()
```

```
➜  mathbot ./mathbot.py
[+] Opening connection to 195.154.53.62 on port 1337: Done
Running..........................................................................................................................................................................................................................................................
Flag is:  ALEXCTF{1_4M_l33t_b0t}
[*] Closed connection to 195.154.53.62 port 1337
```