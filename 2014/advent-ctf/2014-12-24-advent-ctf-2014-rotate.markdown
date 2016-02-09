### Solved by barrebas

A crypto challenge! We're given a zip-file, containing `flag.jpg.enc` and `rotate.py`.

`rotate.py` handles the encryption, but only does so one way:

```python
import sys
import math
import struct

p = lambda x: struct.pack('f', x)
u = lambda x: struct.unpack('b', x)[0]

if len(sys.argv) != 3:
    sys.exit(1)

filename = sys.argv[1]
key = math.radians(int(sys.argv[2]))

bs = open(filename, 'rb').read()
enc = open(filename + '.enc', 'wb')

for i in range(0, len(bs), 2):
    x, y = u(bs[i]), u(bs[i+1])
    enc.write(p(x * math.cos(key) - y * math.sin(key)) + p(x * math.sin(key) + y * math.cos(key)))
```

The script takes two bytes at a time and transforms them into two floats via some goniometric functions, using a key. These floats are then dumped as DWORDS into the encrypted output file. Our first job is to determine what key was used. 

Assuming the flag really **is** a JPEG, we have some prior knowledge. Namely, the first two bytes are `0xff,0xd8`. I made a small file that contains only these two bytes and I ripped the first 8 bytes from `flag.jpg.enc`. I encoded the small 2-byte file with all possible keys and then compared the md5sum of each file to the md5sum of the 8-byte encoded file:

```bash
bas@tritonal:~/adventctf/rotate$ for i in `seq 0 360`; do python rotate.py ./header_real2 $i; mv header_real2.enc output$i; done
bas@tritonal:~/adventctf/rotate$ md5sum header_encoded8
ed3f26885ebe2ba3ec0aa61215e4dcf2  header_encoded8
bas@tritonal:~/adventctf/rotate$ md5sum output* |grep ed3f2
ed3f26885ebe2ba3ec0aa61215e4dcf2  output123
```

So I now know the key that was used was `123`. I then started building a decoder. I hit a few snags on the way. It turns out that `rotate.py` doesn't use the literal value of each byte, but instead uses `struct.unpack('b', byte)` to generate a **signed** representation. That threw me off for a good while. 

The decoder first builds a lookup table of each possible two-byte combination. Then, to decode, it reads eight bytes at a time and looks up the corresponding values from the lookup table. If there are no more floats to be read, it catches the error and outputs the decrypted data:

```python
import sys
import math
import struct

p = lambda x: struct.pack('f', x)
u = lambda x: struct.unpack('b', x)[0]

if len(sys.argv) != 3:
    sys.exit(1)

filename = sys.argv[1]
key = math.radians(int(sys.argv[2]))

enc = open(filename, 'rb')

lookup = {}
for k in xrange(256):
    for l in xrange(256):
        x = struct.unpack('b', chr(k))[0]
        y = struct.unpack('b', chr(l))[0]
        
        x1 = p(x * math.cos(key) - y * math.sin(key))
        x2 = p(x * math.sin(key) + y * math.cos(key))
        
        x3 = struct.unpack('<L', x1)[0]
        x4 = struct.unpack('<L', x2)[0]
        
        lookup[ (x3<<32)+x4 ] = (k, l)
        
going = 1
decoded = ""
while going:
    try:
        f1 = struct.unpack('<L', enc.read(4))[0]
        f2 = struct.unpack('<L', enc.read(4))[0]
        
        (x, y) = lookup[ (f1<<32)+f2 ]
        
        decoded += chr(x)
        decoded += chr(y)
    except:
        going = 0
        print decoded
```

It's *horrible* code, but it got the job done:

![](/images/2014/advent/rotate-flag.jpg)

The flag was `ADCTF_TR0t4T3_f4C3`. 

