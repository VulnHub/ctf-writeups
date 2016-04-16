###Solved by barrebas

Rot 1337, Base 64, MD5, Hex, Flag
50

XOT0X2B1YHUvKEUtKxX4KRXhZHFkXkv5ZET4YuGwKHJ=

So, Rot 1337 == 1337 Mod 26 == Rot 11 == 26 -11 == Rot 15

ROT-11: IZE0I2M1JSFgVPFeViI4VCIsKSQvIvg5KPE4JfRhVSU=
ROT-15: MDI0M2Q1NWJkZTJiZmM4ZGMwOWUzMzk5OTI4NjVlZWY=

Trying base64 for both:
ROT-11 results in 

```
echo "IZE0I2M1JSFgVPFeViI4VCIsKSQvIvg5KPE4JfRhVSU=" | base64 -d
!�4#c5%!`T�^V"8T",)$/"�9(�8%�aU%
```

ROT-15 results in 
```
echo "MDI0M2Q1NWJkZTJiZmM4ZGMwOWUzMzk5OTI4NjVlZWY=" | base64 -d 
0243d55bde2bfc8dc09e339992865eef
```

Continuing with ROT-15 and converting MD5 to hex can be done by using http://md5cracker.org/ which is a multi md5 crack engine, which searchs in various databases and rainbow tables to decrypt your md5 hash.

0243d55bde2bfc8dc09e339992865eef converts to "666c61677b7375703372625f6433637279707433727d"

Let's convert from Hex to ASCII
```
echo "666c61677b7375703372625f6433637279707433727d" | xxd -r -p
flag{sup3rb_d3crypt3r}
```

The flag is **flag{sup3rb_d3crypt3r}**
