### Solved by superkojiman

This is day 6 of the Advent CTF 2014 challenge. We're given a python script called paths.py with the following clue: 

> There are many paths, and search for shortest path from start to goal.
> (to, cost)

The python script takes several arguments, of which needs to be the shortest path to get to the destination. Looking at paths.py we see that we get the flag if the sum of all the arguments passed to the script equal to 2014. As it turns out we can guess the shortest paths, and if it's incorrect, the script returns 1. 

If we look at the script, we see that the first node is (96, 65), so let's try 96 as the first path: 

```
koji@ctf-team ~
$ python paths.py 96

koji@ctf-team ~
$ echo $?
0
```

96 is the first correct path. Let's try (64, 99) next:

```
koji@ctf-team ~
$ python paths.py 96 64

koji@ctf-team ~
$ echo $?
1
```

Nope, that didn't work. So let's try the next one, and the next, and the next, until we get a 0: 

```
koji@ctf-team ~
$ python paths.py 96 94

koji@ctf-team ~
$ echo $?
0
```

So the next one is 94. This will take forever to solve manually so let's use a scrip to do it for us: 


```bash
#!/bin/bash
# solveit.sh

i=0
touch.txt
while [[ $i -lt 150 ]]; do
    python paths.py `cat t.txt` $i >/dev/null
    if [[ $? -eq 0 ]]; then
        grep -w $i t.txt > /dev/null 2>&1
        if [[ $? -ne 0 ]]; then
            echo "+ next node: $i"
            echo -n " $i" >> t.txt
            i=0
            continue
        fi
    fi
    i=$(($i + 1))
done
```

This will write the correct paths to t.txt and pass the contents of t.txt to the next call to paths.py until we get the next correct path. At the end of it all, we should get the shortest path in t.txt

```
koji@ctf-team ~
$ ./solveit.sh 
+ next node: 96
+ next node: 94
+ next node: 72
+ next node: 70
+ next node: 69
+ next node: 89
+ next node: 18
+ next node: 46
+ next node: 22
+ next node: 92
+ next node: 79
+ next node: 59
+ next node: 74
+ next node: 97
+ next node: 58
+ next node: 82
+ next node: 35
+ next node: 85
+ next node: 30
+ next node: 87
+ next node: 25
+ next node: 40
+ next node: 41
+ next node: 7
+ next node: 99
```

That's the shortest path. Let's get that flag now:

```
koji@ctf-team ~
$ python paths.py `cat t.txt`
the flag is: ADCTF_G0_go_5hOr7E57_PaTh
```

The flag is **ADCTF_G0_go_5hOr7E57_PaTh**
