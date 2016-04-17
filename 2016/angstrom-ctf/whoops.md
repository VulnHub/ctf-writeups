### Solved by superkojiman

Easy forensics challenge. We're given a oops.img and are told that a file containing the flag was recently deleted. Just mount it in Linux:

```
# mkdir /mnt; mount oops.img /mnt
# ls -l /mnt/oops
total 14
drwxr-xr-x 6 root root 1024 Feb 16 12:50 .
drwxr-xr-x 4 root root 1024 Feb 16 12:50 ..
-rwx------ 1 root root 4096 Feb 16 12:50 ._.com.apple.timemachine.donotpresent
-rwx------ 1 root root    0 Feb 16 12:50 .com.apple.timemachine.donotpresent
drwxr-xr-x 4 root root 1024 Feb 16 12:50 Documents
drwx------ 2 root root 1024 Feb 16 12:50 .fseventsd
drwx------ 4 root root 1024 Feb 16 12:50 .Spotlight-V100
drwx------ 3 root root 1024 Feb 16 12:50 .Trashes
-rwx------ 1 root root 4096 Feb 16 12:50 ._.Trashes
```

Looks like an OS X directory structure. Deleted files in OS X go into .Trashes, so that's a good place to start.

```
# grep -Rw flag *
Files/books/flatland.txt:flag{waw_.Trashes_isnt_useless_after_all}
```

Done!
