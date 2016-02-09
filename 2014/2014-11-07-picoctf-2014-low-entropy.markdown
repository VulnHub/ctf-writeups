### Solved by barrebas

Low Entropy is a 110 point challenge. 

We are given a server to connect to and a pcap file. We need to decrypt the message found in the pcap file. This message was encoded with a private RSA key. 

```bash
Welcome to the Thyrin drop box. Please send your public key and message.
Public key: c20a1d8b3903e1864d14a4d1f32ce57e4665fc5683960d2f7c0f30d5d247f5fa264fa66b49e801943ab68be3d9a4b393ae22963888bf145f07101616e62e0db2b04644524516c966d8923acf12af049a1d9d6fe3e786763613ee9b8f541291dcf8f0ac9dccc5d47565ef332d466bc80dc5763f1b1139f14d3c0bae072725815f
Message: 49f573321bdb3ad0a78f0e0c7cd4f4aa2a6d5911c90540ddbbaf067c6aabaccde78c8ff70c5a4abe7d4efa19074a5249b2e6525a0168c0c49535bc993efb7e2c221f4f349a014477d4134f03413fd7241303e634499313034dbb4ac96606faed5de01e784f2706e85bf3e814f5f88027b8aeccf18c928821c9d2d830b5050a1e
```

The server spits out the product of two primes, which gives us a lot of possible public keys. We need to find the factors `p` and `q`, the factors of the captured public key. If we have those factors, it's game over. Luckily, the server has only 30 primes to choose from. That means that there are 30 * 29 / 2 = 435 possible keys. Let's first grab all these products of `p` and `q` and store them:

```python
#!/usr/bin/python
from socket import *

while (len(products)<30*29):
	s=socket(AF_INET, SOCK_STREAM)
	s.connect(('vuln2014.picoctf.com',51818))

	# banner
	s.recv(1024)

	pq = s.recv(256)
	pq = long("0"+pq, 16)
	
	if pq and not pq in products:
		with open('public_keys', 'a') as f:
			f.write(str(pq)+"\n")
		f.close()
				
		products.add(pq)
	
	print "Now at {} products...".format(len(products))
	s.close()
```

This should grab the 435 unique public keys. Next, we need to find the primes that constitute the captured public key. Basically, we have a list of `a1*a2, a1*a3, a2*a3...` for 30 unique primes. Let's assume the captured public key is derived from the values `a1` and `a2`. Given a large list of other values ax*ay and some math, we can say that there are values that satisfy:

`
(a1*a2 / (a1*a3) * a2*a3 == a2*a2
`

Therefore, we can extract the squared values of each prime! I skipped using `math.sqrt()` and python `float` values, as these do not have the required precision given these extremely large numbers. Let's generate all these squared values from each key we got from the server. Then, divide the square of the public key by each entry. If the result is also in the list of squared primes, then we have a match!

```python
#!/usr/bin/python
    
with open('public_keys') as f:
	data = f.readlines()
	f.close()
	
alice_key = 0xc20a1d8b3903e1864d14a4d1f32ce57e4665fc5683960d2f7c0f30d5d247f5fa264fa66b49e801943ab68be3d9a4b393ae22963888bf145f07101616e62e0db2b04644524516c966d8923acf12af049a1d9d6fe3e786763613ee9b8f541291dcf8f0ac9dccc5d47565ef332d466bc80dc5763f1b1139f14d3c0bae072725815f

public_keys = set([])
for line in data:
	if len(line.strip()):
		public_keys.add(int(line.strip()))
	
squares = set([])	# sets are fast :)
for key_a in public_keys:
	for key_b in public_keys - set([key_a]):
		squares.add((alice_key * key_a) / key_b)
		
for k in squares:
	l = alice_key**2 / k
	if l in squares:
		if alice_key**2 == l*k:	# double-check to prevent rounding errors
			print "[!] Found k={}\nl={}\n".format(k, l)
```

It should return two values for k and l (because k could be l and vice-versa):

```bash
[!] Found k=145636797632612493383437910621935492258871220486831431433846443881029756884131014317442568196356163696603884037401628766885574744524908524694664229202327755975190209777333222305357215356711196812874146485202755534755335009504417851499146840024376285929565498060947342673068738915934424594894642178132393803401
l=127485391417645634265899520100348459764415230949848696681516013917289651283750339673156991958225605417057264644648275442237083380079695308054967054357615028357457990698626856902554884944611614631356998904650004684028810140797701724207511157802310732003918967758266191880635014381653257954124503965122532941561
```

Now, all we have to do is take the square root of these values to get `p` and `q`! Luckily, [Newton's algorithm](http://stackoverflow.com/questions/15390807/integer-square-root-in-python/15391420#15391420) works perfectly for integers. The server gave us the public exponent (2^16+1) so we are all set for decrypting the message:

```python
#!/usr/bin/python

def isqrt(n):
    x = n
    y = (x + 1) // 2
    while y < x:
        x = y
        y = (x + n // x) // 2
    return x

def egcd(a, b):
	if a == 0:
		return (b, 0, 1)
	else:
		g, y, x = egcd(b % a, a)
		return (g, x - (b // a) * y, y)

def modinv(a, m):
	g, x, y = egcd(a, m)
	if g != 1:
		raise Exception('modular inverse does not exist')
	else:
		return x % m
		
e = 2**16+1
k = 127485391417645634265899520100348459764415230949848696681516013917289651283750339673156991958225605417057264644648275442237083380079695308054967054357615028357457990698626856902554884944611614631356998904650004684028810140797701724207511157802310732003918967758266191880635014381653257954124503965122532941561
l = 145636797632612493383437910621935492258871220486831431433846443881029756884131014317442568196356163696603884037401628766885574744524908524694664229202327755975190209777333222305357215356711196812874146485202755534755335009504417851499146840024376285929565498060947342673068738915934424594894642178132393803401

p = isqrt(k)
q = isqrt(l)
c = 0x49f573321bdb3ad0a78f0e0c7cd4f4aa2a6d5911c90540ddbbaf067c6aabaccde78c8ff70c5a4abe7d4efa19074a5249b2e6525a0168c0c49535bc993efb7e2c221f4f349a014477d4134f03413fd7241303e634499313034dbb4ac96606faed5de01e784f2706e85bf3e814f5f88027b8aeccf18c928821c9d2d830b5050a1e

n = p*q
tot = (p - 1) * (q - 1)
d = modinv(e, tot)

m = pow(c, d, n)
print hex(m)
```

This gives us the decoded hexadecimal representation of the message. Running it through `xxd`:

```bash
bas@tritonal:~/tmp/picoctf/low_entropy$ python solve.py 
0x476f6f64207468696e67206e6f206f6e652063616e207265616420746869732120492764206861746520666f72207468656d20746f206b6e6f7720746861742074686520666c6167206973206d616b655f737572655f796f75725f726e675f67656e6572617465735f6c6f7473615f7072696d65732eL
bas@tritonal:~/tmp/picoctf/low_entropy$ xxd -r -p
476f6f64207468696e67206e6f206f6e652063616e207265616420746869732120492764206861746520666f72207468656d20746f206b6e6f7720746861742074686520666c6167206973206d616b655f737572655f796f75725f726e675f67656e6572617465735f6c6f7473615f7072696d65732e
Good thing no one can read this! I'd hate for them to know that the flag is make_sure_your_rng_generates_lotsa_primes.
```

The flag is `make_sure_your_rng_generates_lotsa_primes`.
