---
layout: post
title: "Angrybird"
date: 2017-02-10 04:59:29 -0500
comments: true
categories: ctf
---

## Solved by superkojiman

angrybird is a reverse engineering challenge. A quick glance at its disassembly in Binary Ninja shows that it prompts the user for the flag and runs a variety of checks on it to verify if the flag is correct. As the challenge name hints, the way to solve it is to use [Angr](http://angr.io/)

Attemping to run the binary causes nothing to happen. That's because before it even asks for input, it checks several things; most of which will cause the binary to exit outright. 

![](/images/2017/codegate/angrybird/01.png)

Right from the get go, we see that `main()` calls `exit()` when EAX is 0. Since EAX is XOR'd with itself, it will always be 0. We can patch this so that it jumps to the desired code path. However, three more tests happen here before `fgets()` is called to prompt us for input:

![](/images/2017/codegate/angrybird/02.png)

The first check `return_21()` expects the function to return 21, but instead it returns 1. The second check `stack_check()` attempts to dereference a non-existent address which causes a segfault. The third check compares the string "hello" with the address of `__libc_start_main()`. All of these can be patched so that we can eventually get to the call to `fgets()` when doing dynamic analysis within gdb.

Once we get to the call to `fgets()`, we're prompted for a flag of 21 bytes. A bunch of `if-else` checks are then done to the input, and if it fails, it goes to a code path that looks like this:

![](/images/2017/codegate/angrybird/03.png)

It basically boils down to this pseudocode: 

```
if input passes check 1 {
    if input passes check 2 {
        if input passes check 3 {
            if input passes check N {
                puts "win!"
            } else {
                puts "melong"
                exit
            }
        } else {
            puts "melong"
            exit
        }
    } else {
        puts "melong"
        exit
    }
}
```

So obviously we want to avoid the branch leading to `exit()` and get to the win branch instead. Angr is a great tool for doing this. 

I needed three things; an address to start at, the address to look for, and addresses to avoid. Angr lets us start anywhere we want in the binary. In this case, I chose to start at 0x4007c2, which is the setup to `fgets()`'s parameters. 

The address to look for is at 0x00404fbc, which prints out the flag we entered. 

![](/images/2017/codegate/angrybird/04.png)

Finally, I needed a list of paths to avoid. For that I used `objdump` to search for 0x00405094; the location of the string "melong".

```
objdump -D angrybird-p4 | grep 0x405094 | awk '{print $1}' | tr -d ":" | sed -e 's/^/0x/g' > avoid.txt
```

With that done, I created my angr script:

```
#!/usr/bin/env python

import angr

def main():
    to_avoid = () 
    with open("avoid.txt") as f: 
        to_avoid = f.readlines()
        to_avoid = [x.strip() for x in to_avoid] 
        to_avoid = [int(x, 16) for x in to_avoid] 

    start_at = 0x4007c2
    success  = (0x404fbc,)
    fail = tuple(to_avoid)

    print "Start at: 0x%X" % (start_at,)
    print "Look for: 0x%X" % (success[0],)
    print "Avoiding %d paths" % (len(fail))

    p = angr.Project("./angrybird", load_options={'auto_load_libs': False})
    state = p.factory.blank_state(addr=start_at)

    path = p.factory.path(state)
    pathgroup = p.factory.path_group(path)

    print "Searching..."
    found_state = pathgroup.explore(find=success, avoid=fail)

    if found_state.found:
        f = found_state.found[0].state

        #import IPython; IPython.embed()
        return "Flag: %s" % (f.posix.dumps(0)[0:21],)
    else:
        return "Sad panda :-("

if __name__ == "__main__":
    #angr.path_group.l.setLevel("DEBUG")
    print main()
```

Running the script slingshots us to the flag: 

![](/images/2017/codegate/angrybird/05.png)
