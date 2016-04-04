### Solved by z0rex

This was stage 1 of a multi level reversing challenge. I started by running the
binary.

```
# ./stage1.bin 
Usage: ./stage1.bin <pass>
# ./stage1.bin password
Try again...
```

So, it was excepting a password as its only argument, and since this was only a
50 point reversing challenge I figured looking at the strings probably wouldn't
be a bad place to start looking.

```
# rabin2 -z stage1.bin
.
.
vaddr=0x0040e079 paddr=0x0000e079 ordinal=4720 sz=26 len=25 section=.rodata type=ascii string=Much_secure__So_safe__Wow
.
.
```

That looks interesting. I tried it as the password and it worked. 

```
# ./stage1.bin Much_secure__So_safe__Wow
Good good!
c3RhZ2UyLmJpbgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAwMDA3NTUAMDAwMTc1
MAAwMDAxNzUwADAwMDAwMTA0MjQwADEyNjU1MTAwNzc2ADAxMjMwMgAgMAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB1c3RhciAgAGpoZXJ2ZQAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAamhlcnZlAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
.
.
.
```

I tried submitting the password as the flag, and it got validated.

It complimented my job and provided me with a massive wall of text which
obviously fits the format of a base64 encoded string.

I copied the string into a file and decoded it. This just threw tons of what
appeared to be garbage at me. So I checked it with the `file` command. 

```
# cat stage1.b64 | base64 -d | file -
/dev/stdin: POSIX tar archive (GNU)
```
Turns out it was a tar archive, which, when extracted, gave me the binary for
the next stage.

```
# cat stage1.b64 | base64 -d | tar xvf -
stage2.bin
```

Flag: `Much_secure__So_safe__Wow`