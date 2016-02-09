### Solved by superkojiman

Decompile with [uncompyle](https://github.com/gstarnberger/uncompyle) and we see this:

```python
if ord(password[1]) != 52 or ord(password[3]) != 36 or ord(password[0]) != 112 or ord(password[7]) != 100 or ord(password[6]) != 114 or ord(password[2]) != 115 or ord(password[5]) != 48 or ord(password[4]) != 119:
    return False
```

Map the numbers to chr(): 

```
>>> map(chr, [52, 36, 112, 100, 114, 115, 48, 119])
['4', '$', 'p', 'd', 'r', 's', '0', 'w']
```

If we take a close look, we see that it's actually "password" in leetspeak.

```
koji@pwnbox32:~/Desktop/pychal$ python i_see_python.py 
Password: p4s$w0rd
tjctf{python_is_fun}
```
The flag is tjctf{python_is_fun}

