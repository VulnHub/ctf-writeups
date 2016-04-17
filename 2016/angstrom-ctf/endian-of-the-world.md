### Solved by z0rex

"Endian of the World" was a binary challenge worth 40 points.

> The end of the world is nigh! Dr. Doomsday has created an evil contraption to
> destroy the planet, and only a single password can stop it! We were able to
> recover the source code for the password check. Find the shortest password
> that will stop Dr. Doomsday's machine and save the world! The program is
> available on the shell server at /problems/endian_of_the_world/, and the binary
> and source are provided. 

In this challenge we're given the source code, which makes life a lot easier for
us. Let's have a look at it before we continue.

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv)
{
    printf("Since you're about to die anyway, I might as well tell you my secret!\n");
    printf("A single password will halt my machine, but you'll never figure it out!\n\n");

    size_t password_length = sizeof(uint32_t)*15 + 2;
    char *buffer = calloc(1, password_length);

    printf("Secret password: \n");
    fgets(buffer, password_length, stdin);
    uint32_t *ints = (uint32_t*) buffer;

    if (ints[0]  == 0x6f645f61 && ints[1]  == 0x64736d6f && ints[2]  == 0x645f7961 && 
        ints[3]  == 0x63697665 && ints[4]  == 0x73695f65 && ints[5]  == 0x6c6e6f5f &&
        ints[6]  == 0x73755f79 && ints[7]  == 0x6c756665 && ints[8]  == 0x5f66695f &&
        ints[9]  == 0x72657665 && ints[10] == 0x656e6f79 && ints[11] == 0x6f6e6b5f &&
        ints[12] == 0x615f7377 && ints[13] == 0x74756f62 && ints[14] == 0x0a74695f)
    {
        printf("Grr, I'll be back!\n");
    }
    else
    {
        printf("Mwahaha! You fools will never stop me! How could the answer be\n");
        printf("0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X 0x%08X?\n", 
                ints[0], ints[1], ints[2], ints[3], ints[4], ints[5], ints[6], ints[7], ints[8], ints[9], ints[10], ints[11],
                ints[12], ints[13], ints[14]);
    }

    return 0;
}
```

That's interesting. All the values after `0x` appears to be hex encoded ASCII 
characters...

```
.
.
.
if (ints[0]  == 0x6f645f61 && ints[1]  == 0x64736d6f && ints[2]  == 0x645f7961 && 
    ints[3]  == 0x63697665 && ints[4]  == 0x73695f65 && ints[5]  == 0x6c6e6f5f &&
    ints[6]  == 0x73755f79 && ints[7]  == 0x6c756665 && ints[8]  == 0x5f66695f &&
    ints[9]  == 0x72657665 && ints[10] == 0x656e6f79 && ints[11] == 0x6f6e6b5f &&
    ints[12] == 0x615f7377 && ints[13] == 0x74756f62 && ints[14] == 0x0a74695f)
{
    printf("Grr, I'll be back!\n");
}
.
.
.
```

... so let's play with that idea.

I turned to Python where I started off by making a list of all the values from
the it statement, and removed the `0x`.

Then I looped over the values and decoded them. First I just decoded the exact
string, but the output made absolutely no sense. So I reversed the hex, just
like little endian, which gave me the flag.

```
# python2
Python 2.7.11 (default, Mar 31 2016, 06:18:34) 
[GCC 5.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> strlist = ["6f645f61","64736d6f","645f7961","63697665","73695f65","6c6e6f5f","73755f79","6c756665","5f66695f","72657665","656e6f79","6f6e6b5f","615f7377","74756f62","0a74695f"]
>>> for a in strlist:
...   print "".join(reversed([a[i:i+2] for i in range(0, len(a), 2)])).decode("hex")
... 
a_do
omsd
ay_d
evic
e_is
_onl
y_us
eful
_if_
ever
yone
_kno
ws_a
bout
_it

>>> out = ""
>>> for a in strlist:
...   out += "".join(reversed([a[i:i+2] for i in range(0, len(a), 2)])).decode("hex")
... 
>>> print out
a_doomsday_device_is_only_useful_if_everyone_knows_about_it
```

Flag: `a_doomsday_device_is_only_useful_if_everyone_knows_about_it`