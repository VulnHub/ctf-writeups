### Solved by bitvijays with special thanks to Swappage

In this crypto challenge, we are provided a encrypted output and a code which was used to encrypt it.

>712249146f241d31651a504a1a7372384d173f7f790c2b115f47

Code:
```
#include <stdio.h>
#include <string.h>

int main() {
  char flag[] = "ADCTF_XXXXXXXXXXXXXXXXXXXX";
  int len = strlen(flag);
  for (int i = 0; i < len; i++) {
    if (i > 0) flag[i] ^= flag[i-1];
    flag[i] ^= flag[i] >> 4;
    flag[i] ^= flag[i] >> 3;
    flag[i] ^= flag[i] >> 2;
    flag[i] ^= flag[i] >> 1;
    printf("%02x", (unsigned char)flag[i]);
  }
  return 0;
}
```

We need to reverse the Right shift and the XOR operation to decode the encrypted string. We spent few hours tried to do the maths which was fruitless. Swappage suggested a question on stackoverflow <a href="http://stackoverflow.com/questions/26481573/reversing-xor-and-bitwise-operation-in-python">Reversing XOR and bitwise operation in Python</a>.

We used the small function written by harold. Thanks harold.

```
def gray2binary(x):
    shiftamount = 1;
    while x >> shiftamount:
        x ^= x >> shiftamount
        shiftamount <<= 1
    return x
```

We modified and used the function in our script to crack this:
``` Python
tr = "712249146f241d31651a504a1a7372384d173f7f790c2b115f47"
b = 0
decoded = ""

def gray2binary(x, s):
    shiftamount = s;
    while x >> shiftamount:
        x ^= x >> shiftamount
        shiftamount <<= 1
    return x

for i in range(0, len(str) - 1, 2):
#Converting string to int
    a = int(str[i:i+2],16)
#Reading previous hex string in case of i > 0
    if i > 0:
        b = int(str[i-2:i],16)

    l=a
#Reversing right shift and XOR of 1,2,3,4
    for s in range(1,5):
        l = gray2binary(l,s)

#XORing the last
    l = l ^ b

#Storing the string
    decoded += chr(l)

print decoded
```

***The flag is ADCTF_51mpl3_X0R_R3v3r51n6***
