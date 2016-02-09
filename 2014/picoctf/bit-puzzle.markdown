### Solved by barrebas

The last bastion of PicoCTF! `bitpuzzle` is a 32-bit ELF file. It asks for a string and first checks if the string is 32 bytes long. Then, it chops it up into four-byte chunks and does certain checks on each chunk, effectively constraining the values that are valid. In the end, if the string passes all checks, then that string is the flag. Looking at other flags, this probably limited our characterset to lowercase plus underscore. The first chuck-checking constraint was this:

```bash
EAX: 0x37363534 ('4567')
EBX: 0xffffd21c ("0123456789abcdefABCDEFGHIJKLMNOP")
ECX: 0xffffffde 
EDX: 0x33323130 ('0123')
ESI: 0x0 
EDI: 0x62613938 ('89ab')
EBP: 0xffffd338 --> 0xffffd3b8 --> 0x0 
ESP: 0xffffd200 --> 0xffffd21c ("0123456789abcdefABCDEFGHIJKLMNOP")
EIP: 0x804858e (lea    ebx,[edi+eax*1])
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048582:	mov    edx,DWORD PTR [esp+0x1c]
   0x8048586:	mov    eax,DWORD PTR [esp+0x20]
   0x804858a:	mov    edi,DWORD PTR [esp+0x24]
=> 0x804858e:	lea    ebx,[edi+eax*1]
   0x8048591:	mov    ecx,0x0
   0x8048596:	cmp    ebx,0xc0dcdfce
   0x804859c:	jne    0x80485ad
```

So for this first constraint, we see that `var2 + var3 == 0xc0dcdfce`. I stepped through the code and set `ebx` to the right value, allowing me to see the other checks. I wrote them down:

```
((b+c) & 0xffffffff) == 0xc0dcdfce
((a+b) & 0xffffffff) == 0xd5d3dddc
((a*3+b*5) & 0xffffffff) == 0x404a7666
((a ^ d) & 0xffffffff) == 0x18030607
((a & d) & 0xffffffff) == 0x666c6970
((b * e) & 0xffffffff) == 0xb180902b
((c * e) & 0xffffffff) == 0x3e436b5f
((e * f*2) & 0xffffffff) == 0x5c483831
((f & 0x70000000) & 0xffffffff) == 0x70000000
((f / g) & 0xffffffff) == 1
((f % g) & 0xffffffff) == 0xe000cec
((e*3+i*2) & 0xffffffff) == 0x3726eb17
((c*4+i*7) & 0xffffffff) == 0x8b0b922d
((d+i*3) & 0xffffffff) == 0xb9cf9c91
```

What else can we infer? Well, since `a+b == 0xd5...`, `0xd5 - first char of a` must be within the ASCII range too. This allowed me to narrow down the values that were possible for the first three chunks of the flag. After messing with `python-constraint`, I started writing my own script. This first script uses python sets, and used just shy of 6 GB of memory; way too much for my poor laptop. I re-wrote the code to use just integer arithmetic. Below is the final script. It's super ugly, but it's late and I'm tired!

```python
#!/usr/bin/python
import itertools, string

def h(i):
	s = hex(i)[2:]
	z = ""
	z += chr(int(s[6:8], 16))
	z += chr(int(s[4:6], 16))
	z += chr(int(s[2:4], 16))
	z += chr(int(s[0:2], 16))
	return z
	
def strToInt(s):
	i = 0
	for x in s:
		i <<= 8
		i += ord(x)
	return i

print "[+] Generating potential strings..."
sz = [x for x in map(''.join, itertools.product("_"+string.lowercase, repeat=4))]

print "[+] Converting potential strings to ints..."
allowed = []
for s in sz:
	allowed.append(strToInt(s))



print "[+] Brutef... Erm, Constraining..."

for s1 in allowed:
	s2 = 0xd5d3dddc - s1
	# OK, so s2+s1 = 0xd0... so we know that that x1 = 0xd0-x2
	# For s1 starting with 61..7a (a-z), 0xd0 - x1 is 0x5b..0x74. 
	# For s1 starting with 41..5a and 30..39, 0xd0 - x1 becomes too large!
	# Therefor, s2 must start with 0x5B .. 0x74 inclusive. 
	if (s2 >= 0x5B000000) and (s2 < 0x75000000):
		s3 = 0xc0dcdfce - s2
		# The same applies for s3
		if (s3 >= 0x4c000000) and (s3 < 0x66000000):
			# This is another constraint
			if ( ((s1 * 3)&0xffffffff) + ((s2*5)&0xffffffff) & 0xffffffff) == 0x404a7666:
				s4 = 0x18030607 ^ s1
				if s4 in allowed:
					print "s1: {}".format(hex(s1))
					print "s2: {}".format(hex(s2))
					print "s3: {}".format(hex(s3))
					print "s4: {}".format(hex(s4))
					for s5 in allowed:
						if ((s5 * s2)&0xffffffff) == 0xb180902b:
							if ((s5 * s3)&0xffffffff) == 0x3e436b5f:
								print "s5: {}".format(hex(s5))
								for s6 in allowed:
									if (s6 & 0x70000000) == 0x70000000:
										#problem.addConstraint(lambda e, f: ((e * f*2) & 0xffffffff) == 0x5c483831, ("e", "f"))
										#problem.addConstraint(lambda f: ((f & 0x70000000) & 0xffffffff) == 0x70000000, ("f"))
										if ((s5 + ((s6 * 2)&0xffffffff))&0xffffffff) == 0x5c483831:
											print "s6: {}".format(hex(s6))
											for s7 in allowed:
												if ((s6 / s7) & 0xffffffff) == 1:
													if ((s6 % s7) & 0xffffffff) == 0xe000cec:
														print "s7: {}".format(hex(s7))
														for s8 in allowed:
															#((e*3+i*2) & 0xffffffff) == 0x3726eb17, ("e", "i"))
															#((c*4+i*7) & 0xffffffff) == 0x8b0b922d, ("c", "i"))
															#((d+i*3) & 0xffffffff) == 0xb9cf9c91, ("d", "i"))
															if (((s5 * 3)&0xffffffff)+((s8*2)&0xffffffff) & 0xffffffff) == 0x3726eb17:
																if (((s3 * 4)&0xffffffff)+((s8*7)&0xffffffff) & 0xffffffff) == 0x8b0b922d:
																	if (((s4 * 1)&0xffffffff)+((s8*3)&0xffffffff) & 0xffffffff) == 0xb9cf9c91:
																		print "s8: {}".format(hex(s8))
																		print "The flag is: "+h(s1)+h(s2)+h(s3)+h(s4)+h(s5)+h(s6)+h(s7)+h(s8)
																		exit(0)

```

The flag is `The flag is: solving_equations_is_lots_of_fun`. That was the last challenge done!
