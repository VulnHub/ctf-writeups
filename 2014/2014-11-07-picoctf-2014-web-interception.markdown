### Solved by barrebas and Swappage

Web Interception is a 140 point challenge. 

`We were able to get some code running in a Daedalus browser. 
Unfortunately we can't quite get it to send us a cookie for its internal 
login page ourselves... But we can make it make requests that we can see, 
and it seems to be encrypting using ECB mode. See here for more details 
about what we can get. It's running at vuln2014.picoctf.com:65414. 
Can you get us the cookie?`

Swappage and me solved this one. The program does the following:

```python
def oracle(s):
	# so, this is simulated. In reality we'd have to run javascript on a target web browser
	# and capture the traffic. That's pretty hard to do in a way that scales, though, so we
	# simulate it instead.
	# This uses ECB mode.
	return AESCipher(key).encrypt(pkcs7_pad('GET /' + s.decode('hex') + secret_data))
```

It takes user-supplied *hex-encoded* data and then returns an encrypted ciphertext in ECB mode. We need to find secret_data. The trick here is that ECB mode will return identical ciphertexts for identical plaintexts. Furthermore, it is a block-mode encryption. This means that we can create a block, have it encrypted, and check if it is present in the returned ciphertext. If so, our block must have matched part of the secret_data. In terms of blocks, we do the following:

```
[ GET /aaaaaaaaaaa                  ] # block one, padded to 16 bytes
[ guess_byte + pkcs7_padding        ] # block two, our guess
[ bogus bytes + part of secret_data ] # block three, part of secret_data
[ ...secret_data...                 ] # block four, more secret_data
[ one byte of secret_data + padding ] # block five, we are interested in the ciphertext of this block!
```

So if the ciphertext of our block two matches that of block five, we know that our guess was ok! We then prepend another byte to our guessed block. We add more padding to block three so that now, not one but *two* bytes of the secret_data are pushed onto block five. We repeat our guess for all possible characters and check the ciphertexts of block two and five. If they match, we have a bingo!

Long story short, here's the script to bruteforce secret_data:

```
#!/usr/bin/python

from socket import *
import string, time

def pkcs7_pad(s):
  l = len(s)
  needed = 16 - (l % 16)
  return s + (chr(needed) * needed)
  
def make_payload(guess):	
	payload = ""
	# pad string to next block
	payload += "a"*11	
	
	# insert guessed bytes, but first pad them
	block = ""
	block += guess
	block = pkcs7_pad(block)
	
	# add guessed bytes + padding
	payload += block				

	# push last bytes into a new ciphertext block. The +1 was empirically determined!
	payload += "a"*(len(guess)+1)	
	
	return payload
	
bruteforce = ""
# think the length of the plaintext is 48 bytes (3 16-byte blocks)
while len(bruteforce) < 48:
	
	# we'll brute-force the entire ascii range
	for in in range(0, 127)
		z = chr(i)
		
		# display progress
		print "[+] trying {}".format(i)
		
		# connect to server
		s=socket(AF_INET, SOCK_STREAM)
		s.connect(('vuln2014.picoctf.com', 65414)) 

		# banner
		time.sleep(0.05)
		s.recv(256)

		# we bruteforce backwards, starting at the last byte
		payload = make_payload(z+bruteforce)
		
		# send payload
		s.send( payload.encode('hex') )

		ciphertext = s.recv(1024)
	
		# split ciphertext into 32 bytes (which are 16 bytes hex-encoded).
		# we need to look for a duplicate block. if we find a duplicate,
		# it means our guessed bytes match the end of the string.
		blocks = [ciphertext[i:i+32] for i in range(0, len(ciphertext), 32)]

		# check if we have the same encoded block as the guess
		# the guessed block is the second returned block, due to the 
		# way the payload is built up. 
		if blocks[1] in blocks[2:]:
			print "match: {}".format(blocks[1])
			bruteforce = z + bruteforce
			print "[+] got {} so far".format(bruteforce)
		s.close()
```

The flag is `congrats_on_your_first_ecb_ecryption\r\n`.
