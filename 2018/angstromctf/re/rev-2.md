### Solved by superkojiman

This challenge has two levels. The first level prompts us for a number. If guessed correctly, we go to the second level where we need to input two two-digit numbers. 

The first number is easy to find. Using Binary Ninja we see the first number compared with the value 4567 in 0x8048546

![](/images/2018/angstromctf/re/rev-2/01.png)

The checks performed on the next two digits aren't quite as simple.

![](/images/2018/angstromctf/re/rev-2/02.png)

This essentially boils down to the following:

```
if x <= 99
    if x > 9
        if y <= 99
            if x < y
                if x * y == 3431
                    win!
```

Based on the above checks, the numbers 47 and 73 match the criteria. The flag is `actf{4567_47_73}`

```
Welcome to Rev2! You'll probably want to use a dissassembler or gdb.
Level 1: What number am I thinking of: 4567
Level 2: Which two two-digit numbers will solve this level. Enter the two numbers separated by a single space (num1 should be the lesser of the two): 47 73
Congrats, you passed Rev2! The flag is: actf{4567_47_73}
```

