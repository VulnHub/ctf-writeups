### Solved by superkojiman

This challenge expects us to input a string that goes through some encoding process. If the encoded result matches a hardcoded string in the binary we win. The expected string can be seen here: 

![](/images/2018/angstromctf/re/rev-3/01.png)

In order to figure out how this string is obtained, we need to look at the `encode()` function. 

![](/images/2018/angstromctf/re/rev-3/02.png)

The instructions highlighted in orange show that each byte in the input string is first XORd with 9, and then 3 bytes are subtracted from it. So it becomes trivial to decode it using:

```
s = "egzloxi|ixw]dkSe]dzSzccShejSi^3q"
d = ""
for c in s:
    x = chr(ord(c) + 3)
    x = chr(ord(x) ^ 9)
    d += x

print d
```

The resulting string is the flag: `actf{reversing_aint_too_bad_eh?}`
