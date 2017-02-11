AlexCTF cr4
---

### Solved by barrebas

A crypto challenge, relating to RSA! Fun times. 

We're given an encrypted message and the public key. The exponent and public parts are extracted like so:

```bash
user@lithium:~/alexctf/cr400$ cat key.out |base64 -d |openssl asn1parse -inform DER -i 
    0:d=0  hl=2 l=  77 cons: SEQUENCE          
    2:d=1  hl=2 l=  13 cons:  SEQUENCE          
    4:d=2  hl=2 l=   9 prim:   OBJECT            :rsaEncryption
   15:d=2  hl=2 l=   0 prim:   NULL              
   17:d=1  hl=2 l=  60 prim:  BIT STRING        
user@lithium:~/alexctf/cr400$ cat key.out |base64 -d |openssl asn1parse -inform DER -i -strparse 17
    0:d=0  hl=2 l=  57 cons: SEQUENCE          
    2:d=1  hl=2 l=  50 prim:  INTEGER           :52A99E249EE7CF3C0CBF963A009661772BC9CDF6E1E3FBFC6E44A07A5E0F894457A9F81C3AE132AC5683D35B28BA5C324243
   54:d=1  hl=2 l=   3 prim:  INTEGER           :010001

```

or, analogously:
```bash
user@lithium:~/alexctf/cr400$ openssl rsa -pubin -inform PEM -text -noout < key.pub
Public-Key: (399 bit)
Modulus:
    52:a9:9e:24:9e:e7:cf:3c:0c:bf:96:3a:00:96:61:
    77:2b:c9:cd:f6:e1:e3:fb:fc:6e:44:a0:7a:5e:0f:
    89:44:57:a9:f8:1c:3a:e1:32:ac:56:83:d3:5b:28:
    ba:5c:32:42:43
Exponent: 65537 (0x10001)
```

[factordb](https://factordb.com/index.php?query=833810193564967701912362955539789451139872863794534923259743419423089229206473091408403560311191545764221310666338878019) makes quick work of this public key, yielding the two prime numbers from which it is constructed. 

```python

from Crypto.PublicKey import RSA
import gmpy

q = long(965445304326998194798282228842484732438457170595999523426901)
p = long(863653476616376575308866344984576466644942572246900013156919)
n = long(833810193564967701912362955539789451139872863794534923259743419423089229206473091408403560311191545764221310666338878019)
e = long(0x10001)	# the exponent

d = long(gmpy.invert(e,(p-1)*(q-1)))
key = RSA.construct((n,e,d,p,q))

print key.exportKey()	# print reconstructed private key!
with open('flag.b64') as f:
	flag = f.readline().decode('base64')
	f.close()

print key.decrypt(flag)
```

```bash
user@lithium:~/alexctf/cr400$ python key.py |strings
-----BEGIN RSA PRIVATE KEY-----
MIH5AgEAAjJSqZ4knufPPAy/ljoAlmF3K8nN9uHj+/xuRKB6Xg+JRFep+Bw64TKs
VoPTWyi6XDJCQwIDAQABAjIzrQnKBvUPnpCxrK5x85DWuS8dbTtmFP+HEYHE3wja
TF9QEkV6ZDCUBers1jQeQwJ5MQIaAImWgwYMdrnA3lgaaeDqnZG+0Qcb6x2SSjcC
GgCZzedK7e6Hrf/daEy8R451mHC08gaS9lJVAhlmZEB1y+i/LC1L27xXycIhqKPe
aoR6qVfZAhlbPhKLmhFavne/AqQbQhwaWT/rqHUL9EMtAhk5pem+TgbW3zCYF8v7
j0mjJ31NC+0sLmx5
-----END RSA PRIVATE KEY-----
ALEXCTF{SMALL_PRIMES_ARE_BAD}
```

Useful links
---

[https://stackoverflow.com/questions/3116907/rsa-get-exponent-and-modulus-given-a-public-key](https://stackoverflow.com/questions/3116907/rsa-get-exponent-and-modulus-given-a-public-key)
[https://warrenguy.me/blog/regenerating-rsa-private-key-python](https://warrenguy.me/blog/regenerating-rsa-private-key-python)
[https://www.dlitz.net/software/pycrypto/api/current/Crypto.PublicKey.RSA-module.html#construct](https://www.dlitz.net/software/pycrypto/api/current/Crypto.PublicKey.RSA-module.html#construct)
