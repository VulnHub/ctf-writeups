### Solved by superkojiman

Function Address is a 60 point reverse engineering challenge. 

> We found this program file on some systems. But we need the address of the 'find_string' function to do anything useful! Can you find it for us?

This is basically a free 60 points. We can get the function address using gdb:

```
# gdb ./problem -q -batch -n -ex 'p find_string'
$1 = {<text variable, no debug info>} 0x8048444 <find_string>
```

The flag is **08048444**
