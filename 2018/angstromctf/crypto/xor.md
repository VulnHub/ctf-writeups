### Solved by guest player RythmStick

```
We found these mysterious symbols hidden in ancient (1950s-era) ruins. We think a single byte may be key to unlocking the mystery. Can you help us figure out what they mean?
```

We're given the string `fbf9eefce1f2f5eaffc5e3f5efc5efe9fffec5fbc5e9f9e8f3eaeee7` to decode.

```
a="fbf9eefce1f2f5eaffc5e3f5efc5efe9fffec5fbc5e9f9e8f3eaeee7"
hx=0

binary_a = a.decode("hex")

def xor_strings(xs, ys):
	return "".join(chr(ord(x) ^ ord(y)) for x, y in zip(xs, ys))


while hx!=255:

	b=str(hex(hx))[2:] * 28
	binary_b = b.decode("hex")
    
	try:
		xored = xor_strings(binary_a, binary_b).encode("ascii")
		if "actf" in xored:
			print xored

	except Exception:
		pass

	hx=hx+1
```

```
➜  ~ python xor.py 
actf{hope_you_used_a_script}
```