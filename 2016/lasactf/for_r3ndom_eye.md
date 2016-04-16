### Solved by superkojiman

Another forensics challenge, this time a PNG file is provided. Looking at the hexdump revealed that there was another PNG file hidden away inside:

```
# xxd -g1 eyeofthetiger.png
.
.
.
001fa870: 22 bd ff 00 2f e9 ff 00 5e 76 f3 3f ff d9 89 50  ".../...^v.?...P
001fa880: 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52 00 00  NG........IHDR..
001fa890: 03 89 00 00 01 cc 08 02 00 00 00 f7 46 7a f4 00  ............Fz..
001fa8a0: 00 00 01 73 52 47 42 00 ae ce 1c e9 00 00 00 04  ...sRGB.........
001fa8b0: 67 41 4d 41 00 00 b1 8f 0b fc 61 05 00 00 00 09  gAMA......a.....
001fa8c0: 70 48 59 73 00 00 0e c3 00 00 0e c3 01 c7 6f a8  pHYs..........o.
.
.
.
```

I wrote a python script to extract it:

```python
#!/usr/bin/env python

offset = 2074750

o = open("out.png", "w")
f = open("eyeofthetiger.png", "r")
f.seek(offset)

o.write(f.read())
```

And running it:

```
# ./carve.py
# file out.png
out.png: PNG image data, 905 x 460, 8-bit/color RGB, non-interlaced
```

Opening it revealed the following image: 

![](/images/2016/lasactf/r3ndom_eye/out.png)

The flag is: `lasactf{rip_my_curly_braces}`
