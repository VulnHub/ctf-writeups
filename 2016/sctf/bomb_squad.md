### Solved by superkojiman and Swappage

It's a classic bomb style reversing challenge. A quick analysis on the binary shows that it has 4 phases that need to be bypassed. 

#### Phase 1

We basically solve X for the following equation:  `1337 = 3 * (2 * X / 37 - 18) - 1`
The answer is 8584.

#### Phase 2

Next we need to provide an array of 6 numbers. These need to be submitted in the format `[ n1, n2, n3, n4, n5, n6 ]`. The first number needs to be 1, as we can see from the assembly:

```
   0x08048aae <+98>:    call   0x8048610 <sscanf@plt>
   0x08048ab3 <+103>:   mov    eax,DWORD PTR [ebp-0x20]
   0x08048ab6 <+106>:   cmp    eax,0x1
```

It then takes the number in our array, passes it to func2(), and then compares the result with a value in ebx.

```
   0x8048ae0 <phase_2+148>:     mov    DWORD PTR [esp],eax
   0x8048ae3 <phase_2+151>:     call   0x8048a22 <func2>
   0x8048ae8 <phase_2+156>:     cmp    ebx,eax
```

If they're the the same, then we lose. So what does func2() do? Something like this:

```c
int func2(int arg) {
    int x = 1;
    for (int i = 0; i < arg; i++) {
        x = x * 2
    }
    return x;
}
```

However, it was easy to just set a breakpoint at 0x8048ae8, and check what ebx was expeting eax to be after func2(), and just *guess* what the number might be. Took a couple of minutes and I figured out phase 2's solution was `[1, 1, 3, 5, 11, 21]`. 

#### Phase 3

Now things get interesting. We basically need to provide a string to solve this phase. Each character in our input string has 0x61 subtracted from it. The resulting value is used as an index to get a character from the keys[] character array. Here's what the keys[] array looks like:

```
keys = [ "q", "a", "g", "v", "C", "Y", "i", "h", "e", "X", "u", "l", "r", "p", "s", "z", "N", "L", "w", "M", "t", "o", "d", "b", "V", "x" ]
```

There's another character array, we'll call it `s`, that has the following value:

```
s = "rqzzepiwMLepiwYsLYtpqpvzLsYeM"; 
```

If the value we got the correct index, and the character we pull from keys is equal to the current character in s, then the current character in our input is correct. Here's the python script I used to solve this:

```python
# calculate phase 3.
# each char in our input string has 0x61 ('a') subtracted from it.
# this value is used as the index to get a char in keys.
# if obtained char from index == current char in s, we got it right.
keys = [ "q", "a", "g", "v", "C", "Y", "i", "h", "e", "X", "u", "l", "r", "p", "s", "z", "N", "L", "w", "M", "t", "o", "d", "b", "V", "x" ]
s = "rqzzepiwMLepiwYsLYtpqpvzLsYeM"
win = ""
for i in s:
    c = 97 + keys.index(i)
    win += chr(c)

buf += win
```

In the end, this prints out `mappingstringsforfunandprofit`.

#### Phase 4
Swappage figured out that we need to enter 7 numbers, where each number can have the value 1, 2, or 3. I took a quick peek at the function, but didn't bother to figure out what it did. Instead, I used a python constraint solver to just brute force it. I already knew how to solve phase 1, 2, and 3, so I used pwntools to just load up the bomb_squd binary and feed it the inputs and guesses to phase 4: 

```python
#!/usr/bin/env python
from constraint import *
from pwn import *
import time

# phase 4
problem = Problem()
min = 1
max = 4

problem.addVariables(["a", "b", "c", "d", "e", "f", "g"], range(min, max))
n = len(problem.getSolutions())
print "Possible solutions to phase 4:", n
time.sleep(5)

for i in range(0, n):
    sh = process("./bomb_squad")
    print sh.recvline()
    print sh.recvline()

    # solve phase 1
    sh.sendline("8584")
    print sh.recvline()

    # solve phase 2
    sh.sendline("[1, 1, 3, 5, 11, 21]")
    print sh.recvline()
    print sh.recvline()

    # solve phase 3
    sh.sendline("mappingstringsforfunandprofit")
    print sh.recvline()

    d = problem.getSolutions()[i]
    s = "%d %d %d %d %d %d %d" % (
        d['a'],
        d['b'],
        d['c'],
        d['d'],
        d['e'],
        d['f'],
        d['g'])
    print "iter %d, sending: %s" % (i, s)
    sh.sendline(s)
    r = sh.recvline()
    print r
    if "flag" in r:
        print "Solved phase 4:", s
        break

    sh.close()
print "Script completed."
```

The answer was `1 1 2 2 1 1 3`. And now we could get the flag:

```
# cat in.txt
8584
[1, 1, 3, 5, 11, 21]
mappingstringsforfunandprofit
1 1 2 2 1 1 3
# cat in.txt | nc problems2.2016q1.sctf.io 1340
Welcome to the bomb squad! Your first task: Diffuse this practice bomb.
Give me a number!
You got through phase 1 alright! Good work! But can you handle phase 2?
Give me an array of numbers!
You could handle it! Good job... I think you can handle phase 3... right?
DAYUM, you got it! You know the drill, time for phase 4.
Congratulations, you won! Here's the flag:
sctf{g00d_j0b_c4d3t}
```
