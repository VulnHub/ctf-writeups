### Solved by Swappage

Math, math, and more math! :)

There was a loth of math in this picoCTF, and Tick Tock was a pretty cool one.

The problem was under the reverse engineering category, but it was definitely more math related then reverse engineering, as all you had to understand in terms of reversing, was what the python script was doing.

<!-- more -->

If we remove all the visualization stuff, for spinning the clock all the problem revolves around these functions

```python
	def count(n,m,msg=""):
	  # n % m
	  spin_for = min((m*10),n)
	  nspots = 112
	  for i in xrange(0,spin_for,m/nspots+1):
		speed = 1.0/((spin_for-i)/(m/nspots+1)+1)
		printc(clock(1+i+n-spin_for,m),msg)
		time.sleep(speed)
	  if m/nspots+1 > 2:
		for j in xrange(i,spin_for,max((spin_for-i)/50,1)):
		  printc(clock(1+j+n-spin_for,m),msg)
	  printc(clock(n,m),msg)
	  return n%m
```

the count() function: that is nothing but a modulus operation with some ascii art :)

```python
	def powmod(n,p,m,msg=""):
	  # n^p % m
	  for i in xrange(max(0,p-100),p+1):
		speed = 1.0/math.sqrt(1+p-i)
		printc(clock(pow(n,i,m),m),msg)
		time.sleep(speed)
	  return pow(n,p,m)
```

and the powmod() function: which is nothing but a pow, again with some graphics for displaying the clock.

then the application does the following:

it takes the first argument number we supply, it takes the list named secretz, containing 17 tuples

```python
	secretz = [(1, 2), (2, 3), (8, 13), (4, 29), (130, 191), (343, 397), (652, 691), (858, 1009),
           (689, 2039), (1184, 4099), (2027, 7001), (5119, 10009), (15165, 19997), (15340, 30013),
           (29303, 70009), (42873, 160009), (158045, 200009)]
```

and cycles over each tuple using the count() function, which as said earlier is nothing but a modulus operation.

```python
	for (r,m) in secretz:
	  if count(num,m,"%d %% %d"%(num,m)) != r:
		print
		print "%d %% %d != %d... WRONG"%(num,m,r)
		sys.exit(0)
	  else:
		print
		print "%d %% %d == %d... GOOD"%(num,m,r)
		time.sleep(2)
		print '\033[2A'
		print " "*90
```

it performs the modulus operation between the number we supplied and the second value of the tuple, and checks that the result of the operation matches the first value, if the result is correct it procedes to the next tuple, if it fails, it exits.

for example, if we assume that x is the first argument we submitted it checks that the following statement is true

$$
\begin{align}
x \equiv 1 \pmod{2}
\end{align}
$$

and if it is, it performs the check for the next tuple.

So the first part of the problem was to find a value x that would satisfy the equivalence for every tuple in the secretz list.

This problem can be solved using the *chinese remainder theorem*
which can be implemented in python as follows:

```python
	#!/usr/bin/python

	def mul_inv(a, b):
		b0 = b
		x0, x1 = 0, 1
		if b == 1: return 1
		while a > 1:
			q = a / b
			a, b = b, a%b
			x0, x1 = x1 - q * x0, x0
		if x1 < 0: x1 += b0
		return x1
	 
	def chinese_remainder(n, a, lena):
		p = i = prod = 1; sm = 0
		for i in range(lena): prod *= n[i]
		for i in range(lena):
			p = prod / n[i]
			sm += a[i] * mul_inv(p, n[i]) * p
		return sm % prod
	 
	if __name__ == '__main__':
		n = [2, 3, 13, 29, 191, 397, 691, 1009, 2039, 4099, 7001, 10009, 19997, 30013, 70009, 160009, 200009]
		a = [1, 2, 8, 4, 130, 343, 652, 858, 689, 1184, 2027, 5119, 15165, 15340, 29303, 42873, 158045]
		c = chinese_remainder(n, a, len(n))
		print c
```

Once we find the number that satisfies the first part of the problem, the second function comes in play

```python
	if powmod(num,num2,200009*160009,"%d ^ %d %% %d"%(num,num2,200009*160009)) != 1:
	  print
	  print "%d ^ %d %% %d != 1... WRONG"%(num,num2,200009*160009)
	  sys.exit(0)
	else:
	  print "Congratulations! The flag is: %s_%s"%(sys.argv[1],str(num2))
```

and what happens here is that the last two modulus operand of the secretz list, are multiplied one another and used as a modulus to compute the following equation

$$
\begin{align}
x^y \equiv 1 \pmod{N}
\end{align}
$$

where 

- x is our first argument
- y is our second argument
- and N is the number obtained by the multiplication said above

Solving this second part is easy, because there is the *Euler's theorem* stating

$$
\begin{align}
a^{\phi(N)} \equiv 1 \pmod{N}
\end{align}
$$

so.. Wolfram Alpha to the rescue!

let's calculate $$\phi(N)$$ as this will give us the correct value to submit to the program as second parameter.

Unfortunately the clock script was buggy, and during the last step returned an exception, but in the end the calculated numbers were correct, and scored the flag successfully on the web site.

	83359654581036155008716649031639683153293510843035531_32002880064
