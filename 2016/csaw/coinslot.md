coinslot
---

The service that we need to connect to gives an amount in dollars, and then asks the appropriate amount of dollar bills and coins. I whipped up a python script that could solve it for me, and after ironing out the bugs, I ran it for a few minutes. The first iteration of the script was using floats, which killed the calculation once large enough values started appearing. I fixed it by converting to cents. I ran the script again, only to find that the script was hanging and not returning the flag. This was probably because it was looking for a string in the response that wasn't there. That's why the Ctrl-c trap was added ;) 

```python
#!/usr/bin/python

from socket import *

def recvtil(x):
	buf = ''
	try:
		while not x in buf:
			buf = buf + s.recv(1)
	except KeyboardInterrupt:
		print buf
	return buf

global s
s = socket(AF_INET, SOCK_STREAM)

s.connect(('misc.chal.csaw.io',8000))

task = s.recv(100).split('\n')
task = task[0]
while (1):
	print task
	amount = int((task[1:]).replace('.',''))
	print amount

	coins = [1000000, 500000, 100000, 50000, 10000, 5000, 2000, 1000, 500, 100, 50, 25, 10, 5, 1]

	for c in coins:
		s.send(str(int(amount / c))+"\n")
		task = recvtil(': ')
		amount = amount - int(amount / c) * c

	task = task.split('\n')
	print task
	task = task[1]
s.close()
```

Yes, it's ugly. Yes, I suck at coding. 

```
['correct!', '$31759.48', '$10,000 bills: ']
$31759.48
3175948
^Ccorrect!
flag{started-from-the-bottom-now-my-whole-team-fucking-here}


['correct!', 'flag{started-from-the-bottom-now-my-whole-team-fucking-here}', '', '']
flag{started-from-the-bottom-now-my-whole-team-fucking-here}
Traceback (most recent call last):
  File "coinslot.py", line 23, in <module>
    amount = int((task[1:]).replace('.',''))
ValueError: invalid literal for int() with base 10: 'lag{started-from-the-bottom-now-my-whole-team-fucking-here}'
```

A quick Ctrl-c once the output of the script stopped yielded the flag:

```flag{started-from-the-bottom-now-my-whole-team-fucking-here}```
