AlexCTF crypto3
---

### Solved by barrebas


Having just solved [crypto4](), solving crypto3 was easy enough, changing the values in the python script.

```python
from Crypto.PublicKey import RSA
import gmpy

q = long(0xfa0f9463ea0a93b929c099320d31c277e0b0dbc65b189ed76124f5a1218f5d91fd0102a4c8de11f28be5e4d0ae91ab319f4537e97ed74bc663e972a4a9119307)
p = long(0xa6055ec186de51800ddd6fcbf0192384ff42d707a55f57af4fcfb0d1dc7bd97055e8275cd4b78ec63c5d592f567c66393a061324aa2e6a8d8fc2a910cbee1ed9)
n = long(q * p)
e = long(0x6d1fdab4ce3217b3fc32c9ed480a31d067fd57d93a9ab52b472dc393ab7852fbcb11abbebfd6aaae8032db1316dc22d3f7c3d631e24df13ef23d3b381a1c3e04abcc745d402ee3a031ac2718fae63b240837b4f657f29ca4702da9af22a3a019d68904a969ddb01bcf941df70af042f4fae5cbeb9c2151b324f387e525094c41)
c = long(0x7fe1a4f743675d1987d25d38111fae0f78bbea6852cba5beda47db76d119a3efe24cb04b9449f53becd43b0b46e269826a983f832abb53b7a7e24a43ad15378344ed5c20f51e268186d24c76050c1e73647523bd5f91d9b6ad3e86bbf9126588b1dee21e6997372e36c3e74284734748891829665086e0dc523ed23c386bb520)


d = long(gmpy.invert(e,(p-1)*(q-1)))
key = RSA.construct((n,e,d,p,q))

print key.exportKey()   # print reconstructed private key!
print hex(key.decrypt(c))[2:-1].decode('hex')
```

Yielding

```bash
user@lithium:~/alexctf/cr3$ python key.py 
-----BEGIN RSA PRIVATE KEY-----
MIIC2wIBAAKBgQCiK1kVca9ODUZXdLgszZlS60ef391VGADClAscEU4lQU/GLGAI
ZWPfbLovwWTK/Q2yUymd3fpX2MToevnyce+eDLEKuFE55c9Srjn/f4MzgPvzQ543
TDHqJrIiSy0YyxlOOzuxPNZAnJeOPo/zV5TtTPjK0DBbKwWy7mbAnKJy7wKBgG0f
2rTOMhez/DLJ7UgKMdBn/VfZOpq1K0ctw5OreFL7yxGrvr/Wqq6AMtsTFtwi0/fD
1jHiTfE+8j07OBocPgSrzHRdQC7joDGsJxj65jskCDe09lfynKRwLamvIqOgGdaJ
BKlp3bAbz5Qd9wrwQvT65cvrnCFRsyTzh+UlCUxBAoGAGveOdgu6T3tV///XufIg
kKAPmsYpaX/XqXG2T4HWo7v2xVLDwpq+WuoHJX5vz9L2MYbaMSUrvgMnhzL4pNj3
81+MvtOKyV9b6Srtvwk0FHUNEay4oAr8mr9NO0lthJQ27xgmwnoD42LtYvdMzEIf
jD5i1rWWdXgiCAiz9isvbHECQQCmBV7Bht5RgA3db8vwGSOE/0LXB6VfV69Pz7DR
3HvZcFXoJ1zUt47GPF1ZL1Z8Zjk6BhMkqi5qjY/CqRDL7h7ZAkEA+g+UY+oKk7kp
wJkyDTHCd+Cw28ZbGJ7XYST1oSGPXZH9AQKkyN4R8ovl5NCukasxn0U36X7XS8Zj
6XKkqRGTBwJAOni6h6EjYzebypMQN7Ft51FsvYsUx23CvGaNTEruj9CS9xY4AOW/
OxzAU/8q3KR/7n0vdIy72IALPxq00JtCyQJBALXiVkEIFmUO3eHtHTUlJB4rkiqi
7Cg77kaJG6+yiND+t+0Aex1+pCLn6p2daIHSTS87IfTrJgyRMFIlIMBKGC8CQQCF
J701Kpjj1UK5PThaVsdwyS8+VKaq+cvjgoQvQeZaS0g/Q6NIhMkb+nLCZ7++Ukc6
FYeFomuDxViarzXNSJJU
-----END RSA PRIVATE KEY-----
ALEXCTF{RS4_I5_E55ENT1AL_T0_D0_BY_H4ND}
```

