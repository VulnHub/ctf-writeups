### Solved by bitvijays

ZOR is a 50 point Cryptography challenge. You are provided with a encrypted file and the password encryption method in ZOR.py

> Daedalus has encrypted their blueprints! Can you get us the password? 
ZOR.py
encrypted

```
#!/usr/bin/python

import sys

"""
Daedalus Corporation encryption script.
"""

def xor(input_data, key):
    result = ""
    for ch in input_data:
        result += chr(ord(ch) ^ key)

    return result

def encrypt(input_data, password):
    key = 0
    for ch in password:
        key ^= ((2 * ord(ch) + 3) & 0xff)

    return xor(input_data, key)

def decrypt(input_data, password):
    return encrypt(input_data, password)

def usage():
    print("Usage: %s [encrypt/decrypt] [in_file] [out_file] [password]" % sys.argv[0])
    exit()

def main():
    if len(sys.argv) < 5:
        usage()

    input_data = open(sys.argv[2], 'r').read()
    result_data = ""

    if sys.argv[1] == "encrypt":
        result_data = encrypt(input_data, sys.argv[4])
    elif sys.argv[1] == "decrypt":
        result_data = decrypt(input_data, sys.argv[4])
    else:
        usage()

    out_file = open(sys.argv[3], 'w')
    out_file.write(result_data)
    out_file.close()

main()
```

If you see the encrypt function, key is bitwise and with 0xFF which means there are only 256 possible key combinations.
```
def encrypt(input_data, password):
    key = 0
    for ch in password:
        key ^= ((2 * ord(ch) + 3) & 0xff)
    return xor(input_data, key)

```
If we add a new function in the given ZOR.py, TEST is added just to distinguish the result of different key decryptions.
```
def sol(input_data):
    result = ""
    for key in range(0,255):
        result += xor(input_data,key)
        result += "  TEST  "
    return result 
```
and call this function in with a new parameter
```
    if sys.argv[1] == "encrypt":
        result_data = encrypt(input_data, sys.argv[4])
    elif sys.argv[1] == "decrypt":
        result_data = decrypt(input_data, sys.argv[4])
    elif sys.argv[1] == "sol":
        result_data = sol(input_data)
    else:
        usage()
```
Further, we ran the script with python ZOR.py sol encrypted res.out 12. 12 is the random just to skip the check of length of parameters.

This creates a res.out file with the results of decryption from all possible keys of 0 to 255. If you manually check this file, you would find the below:
```
This message is for Daedalus Corporation only. Our blueprints for the Cyborg are protected with a password. That password is b19eb1a26991e6ea6365f211ec5437
```
The flag is **b19eb1a26991e6ea6365f211ec5437**

