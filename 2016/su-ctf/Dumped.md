###Solved by bitvijays

***Challenge 2: Dumped 100 Points***: In Windows Task Manager, I right clicked a process and selected "Create dump file". I'll give you the dump, but in return, give me the flag!

In this challenge, we were provided RunMe.DMP.xz, extracting it and checking strings and greping for the flag format provides you the flag.
```
unxz RunMe.DMP.xz 
strings RunMe.DMP | grep Sharif

***SNIP***
00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000SharifCTF{4d7328869acb371ede596d73ce0a9af8}
SharifCTF{4d����7328869acb371ede596ce0aJ�f8}
SharifCTF{4d7328869acb371ede596d73ce0a9af8}
SharifCTF{4d����7328869acb371ede596ce0aJ�f8}
***SNIP***
```

The flag is ***SharifCTF{4d7328869acb371ede596d73ce0a9af8}***
