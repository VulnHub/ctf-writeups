### Solved by superkojiman

Easy Overflow is a 40 point binary exploitaiton challenge. This one's a warm up challenge. 

> Is the sum of two positive integers always positive?
nc vuln2014.picoctf.com 50000
'nc' is the Linux netcat command. Try running it in the shell.

When we connect to the server, we're given a number and we're prompted for a number to make it negative. Eg:

```
$ nc vuln2014.picoctf.com 50000
Your number is 2088572. Can you make it negative by adding a positive integer?
12345
Almost... the sum was 2100917.

Thanks for playing.
```

The solution is to cause an integer overflow. 2,147,483,647 is the maximum positive value for a 32-bit signed integer, so if we enter that it should cause an integer overflow when added to the server's number: 

```
$ nc vuln2014.picoctf.com 50000
Your number is 440902. Can you make it negative by adding a positive integer?
2147483647
Congratulations! The sum is -2147042747. Here is the flag: That_was_easssy!

Thanks for playing.
```

The flag is **That_was_easssy!**
