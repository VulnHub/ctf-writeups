### Solved by superkojiman

For this reversing challenge, I was given a 32-bit PE binary. Although I got the flag for this challenge, I don't believe I solved it in the intended way. I started off using static analysis with IDA and quickly figured out that it expected 4 arguments, all of which had to be numbers. Made sense since the challenge wsa called The Vault, so presumably it was expecting the key to unlock it. Here's how the binary behaved based on the arguments provided:

```
C:\Users\koji\Desktop>source.exe
Epic Fail!

C:\Users\koji\Desktop>source.exe aaaa aaaa aaaa aaaa
Hogwarts

C:\Users\koji\Desktop>source.exe 1 2 3 4
4 digit numbers are the way forward

C:\Users\koji\Desktop>source 1111 2222 3333 4444
So u think you found the flag
```

The last case here was interesting, in that the string "So you think you found the flag", was nowhere to be found in the binary. Perhaps it was being generated on the fly. Looking into it further, I traced the execution to a function called getdigs() at 0x0040151D. It takes a pointer to an int, in this case, the first argument we provided. Looking at the assembly, it was clear that it was performing some math on each of the digits in the four digit argument. I loaded it up in Immunity Debugger,  but before I could set a breakpoint at getdigs(), I noticed something interesting int he dump window:

![](/images/2016/pragyan/vault/01.png)

That's the string "So u think you found the flag", and a little bit more. Right after it was the text "keygen". Since this CTF has no standard flag format, I thought what the hell, and tried "keygen" as the flag and it worked!

Easy peasy?
