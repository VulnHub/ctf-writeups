AlexCTF crypto5
---

### Solved by barrebas

This challenge hinted at a weak number generator. We're given an address and port to connect to: ```nc 195.154.53.62 7412```. Upon connecting, we're presented with the following:

```bash
Guessed 0/10
1: Guess the next number
2: Give me the next number
```
Guessing a wrong number (option 2) immediately disconnects the program. Option 1 gives the next number, as promised. 

I started by grabbing the first 10,000 numbers, hoping for duplicates. There were none! I spent some time looking at breaking linear congruential generators, which I believed to be the number generator in use. In parallel, I downloaded the first 50,000 numbers. The odd thing was that after exactly 32K numbers, numbers started repeating. However, the service itself did not reuse these numbers. I suspect that the lcg is set up each time a user connects, generating a unique set of numbers. After 32K numbers, however, these wrap around. 

As a side note, it [should be possible](https://security.stackexchange.com/questions/4268/cracking-a-linear-congruential-generator) to break the lcg using just three sequential numbers. It was getting late and I was getting sleep, however.  

```python
from socket import *


def recvuntil(x):
	buf = ''
	while not x in buf:
		buf += s.recv(100)

	return buf

global s
import re
s = socket(AF_INET, SOCK_STREAM)
s.connect(('195.154.53.62',7412))
recvuntil('2: Give me the next number')



numbers = []

while True:
	s.send('2\n')
	data = recvuntil('2: Give me the next number')
	needle = re.compile('^[0-9]*$', re.MULTILINE )
	m = re.search(needle, data)
	new_num = int(m.group(0))
	if new_num in numbers:
		for i in range(20):
			print numbers[i + numbers.index(new_num)]	
		import telnetlib
		t = telnetlib.Telnet()
		t.sock = s
		t.interact()
	else:
		print "appending {}".format(new_num)
		print "got {} so far".format(len(numbers))
		numbers.append(new_num)
s.close()
```

and in action (took around 30 minutes to get full circle):

```bash
got 32752 so far
appending 276556517
got 32753 so far
appending 1042045645
got 32754 so far
appending 57023606
got 32755 so far
appending 2023971853
got 32756 so far
appending 492526862

...


appending 2759103407
got 32761 so far
appending 1667899795
got 32762 so far
appending 3881176241
got 32763 so far
appending 1857514483
got 32764 so far
appending 2799781652
got 32765 so far
appending 167224480
got 32766 so far
appending 3772149431
got 32767 so far
417067408
21386687
2508691915
3861506816
1897405663
3820197047
1347627130
3885520901
2227633198
2595838637
3779085340
2727745706
1783375528
810648032
2450954500
342773006
3454113583
794318207
4076865415
1707328526
1
Next number (in decimal) is
21386687
Guessed 1/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
2508691915
Guessed 2/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
3861506816
Guessed 3/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
1897405663
Guessed 4/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
3820197047
Guessed 5/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
1347627130
Guessed 6/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
3885520901
Guessed 7/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
2227633198
Guessed 8/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
2595838637
Guessed 9/10
1: Guess the next number
2: Give me the next number
1
Next number (in decimal) is
3779085340
flag is ALEXCTF{f0cfad89693ec6787a75fa4e53d8bdb5}
```

So the last part had to be done by hand, but went fine the first time around ;)

