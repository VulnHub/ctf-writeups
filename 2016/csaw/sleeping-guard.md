CSAW 2016 Sleeping Guard
---

Upon connecting to the address that is supplied, we receive the following file:

```bash
nc crypto.chal.csaw.io 8000 |base64 -d > sleeping.png
```

And the file:

```
0000000: de3f 0f2f 524b 4541 6579 2132 1e27 053a  .?./RKEAey!2.'.:
0000010: 5f41 5eb7 6579 2197 5f69 4168 5f44 6517  _A^.ey!._iAh_De.
0000020: 2379 213f 5308 0025 1e41 5ffa ea72 dd5e  #y!?S..%.A_..r.^
0000030: 526f 4168 7f22 1719 2879 2145 716f 41e8  RoAh."..(y!EqoA.
0000040: db41 5fb1 6579 21bf bf6f 411d 6f41 5fa1  .A_.ey!..oA.oA_.
0000050: 0579 2105 cf6f 417f 2fdd e51a 5979 213f  .y!..oA./...Yy!?
0000060: 5e1f 0931 2c41 5f59 1179 212d 236e 9f0e  ^..1,A_Y.y!-#n..
...snip...
```

This smells like a XOR encrypted image file. See the first few bytes, where it says 'ey!'? Could these be NULL bytes, encoded with something like 'key!' and divulging the XOR key? 

I guessed the received file would be a PNG, so I took the header of a normal PNG and XORed the first few bytes of the received file with the bytes of the normal PNG. This yielded 'WoAh_A_Key!?' and told me I was on the right track. A small python script later:


```python
#!/usr/bin/python


f = open("sleeping.png").read()

buf = ''
key = 'WoAh_A_Key!?'
for i in range(len(f)):
	buf = buf + chr(ord(f[i]) ^ ord(key[i % 12]))

print buf
```

And this gave the image

![sleeping guard](/images/2016/csaw/sleeping-guard/sleep.png)

With the flag 'flag{l4zy_H4CK3rs_d0nt_g3T_MAg1C_FlaG5}.

