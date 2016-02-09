### Solved by Swappage

RSA Mistakes was a 200 points cryptography master challenge in picoCTF.
The final objective was to recover the encrypted message.

<!-- more -->

![](/images/2014/pico/rsa-mistakes/problem.png)

In the downloaded zip file we were provided with

- a traffic capture file in tcpdump format
- a python script

By looking at the pcap file we could notice that there are a bunch of request/responses, which by the problem statement were most likely encrypted using RSA.

![](/images/2014/pico/rsa-mistakes/tcpstreams.png)

Two of these streams, the first and the fourth, were extremely interesting, because two over the three parameters in the request were identical, while the third was not, and on the other side, the reply, which is supposed to be the encrypted message was completely different.

So, at this point, let's give a look at the server python script and try to figure things out.

```python
	#!/usr/bin/env python
	import os
	from Crypto.PublicKey import RSA
	import SocketServer
	import threading
	import time

	message = ''
	with open('message.txt', 'r') as f:
	  message = f.read()

	class threadedserver(SocketServer.ThreadingMixIn, SocketServer.TCPServer):
	  pass

	id_dict = {}

	class incoming(SocketServer.BaseRequestHandler):
	  def handle(self):
		cur_thread = threading.current_thread()
		welcome = "Welcome to the Daedalus Corp Message Service\n"
		self.request.send(welcome)
		self.request.send("Please send me a public key and an ID. We'll encrypt the message")
		self.request.send(" and send it to you.\n")
		key = self.request.recv(4096)
		print "got %s" % key
		N, e, user_id = key.split(' ')
		print N, e
		N = int(N, 16)
		e = int(e)
		user_id = int(user_id)
		print "Got the user id!"
		encrypted = hex(pow(user_id * int(message.encode('hex'), 16) + (user_id**2), e, N))
		print "encrypted."
		self.request.send(encrypted)
		print "DOne!"

	server = threadedserver(("0.0.0.0", 12345), incoming)
	server.timeout = 4
	server_thread = threading.Thread(target=server.serve_forever)
	server_thread.daemon = True
	server_thread.start()
	server_thread.join()
```

The interesting part is the following:

```python
		N, e, user_id = key.split(' ')
		print N, e
		N = int(N, 16)
		e = int(e)
		user_id = int(user_id)
		print "Got the user id!"
		encrypted = hex(pow(user_id * int(message.encode('hex'), 16) + (user_id**2), e, N))
```

By looking at these instructions we can notice that

- the application recenves 3 parameters as imput, separated by a space
- the program splits them to generate N, e and user_id, where N is the modulus and e is the exponent
- and then it encrypts the message.

But if we look at the encryption function more closely, we can notice that the user_id plays a key role here, as it's used to modify the supplied message before the RSA encryption is performed.

The encryption itself is done with a typical RSA equation where we have

$$
\begin{align}
c &\equiv m^e \pmod{n}
\end{align}
$$

but m in this specific case is

$$
\begin{align}
m = user\_id * t + user\_id^2
\end{align}
$$

where t is the user supplied text

So, putting things togeder we have:

- two different encrypted messages
- a small public exponent: 3
- the same public key being used for both messages, because the public key (e, n) is the same in both encryption processes

Plus, we also have a user_id parameter which differs, and allows us to determine a linear relation between the two messages so that

$$
\begin{align}
m_1 &\equiv \alpha m_2 + \beta \pmod{n} 
\end{align}
$$

This is a convenient situation, because there is an interesting theorem: the theorem of Franklin-Raiter that states:

Set $$ e = 3 $$ and let $$ (N, e)$$  be an RSA public key. Let $$ M_1 \neq M2 \in \mathbb{Z}^*_N $$ satisfy $$ M_1 \equiv f(M_2) \pmod{n} $$ for some linear polynomial $$ f = \alpha x + \beta \in \mathbb{Z}_N[x] $$ with $$ b \neq 0 $$ . Then, given $$ (N, e, C_1, C_2, f) $$, the attacker can recover $$ M_1, M_2 $$ in time quadratic in $$log_2 N $$.

The original paper which can be found here: https://www.cs.unc.edu/~reiter/papers/1996/Eurocrypt.pdf

also provides us with all the mathematical functions we need to calculate the value of our plaintext message from the encrypted messages: so all we were left to do was to calculate $$M_2$$ in function of $$M_1$$ as shown above, and implement the formula in a python script to compute the values for us; once the plaintext for the message is recovered, it's then just a matter of extracting $$t$$.

First let's start by working on the messages and have them set in a proper way: since we need $$\alpha$$ and $$\beta$$ we procede as follows.

Given

- user_id1 as the first message userid
- user_id2 as the second message userid
- t as the plaintext we need to recover
- x as the modular inverse multiplier of 37

$$
\begin{gather*}
user\_id1 = 37 \\
user\_id2 = 52 \\
\\
M_1 \equiv 37 * t + 37^2 \pmod{n} \\
M_2 \equiv 52 * t + 52^2 \pmod{n} \\
\\
t \equiv x(M_1 - 37^2) \pmod{n} \\
\\
m2 \equiv 52xM_1 - 52*37^2x +52^2 \pmod{n} \\
\\
a \equiv 52x \pmod{n} \\
b \equiv 52^2 - 52*37^2x \pmod{n} \\

\end{gather*}

$$

Now that we have $$\alpha$$ and $$\beta$$ what remains to do is punch them in our python script with $$C_1, C_2$$ and $$N$$ to retrieve $$M_1$$

```python
	#!/usr/bin/python

	# Functions for calculating the modular inverse multiplier
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

	#The modulus
	N = "fd2adfc8f9e88d3f31941e82bef75f6f9afcbba4ba2fc19e71aab2bf5eb3dbbfb1ff3e84b6a4900f472cc9450205d2062fa6e532530938ffb9e144e4f9307d8a2ebd01ae578fd10699475491218709cfa0aa1bfbd7f2ebc5151ce9c7e7256f14915a52d235625342c7d052de0521341e00db5748bcad592b82423c556f1c1051"
	N = int(N, 16)

	#The exponent
	e = 3

	#Encrypted message 1
	c1 = "0x81579ec88d73deaf602426946939f0339fed44be1b318305e1ab8d4d77a8e1dd7c67ea9cbac059ef06dd7bb91648314924d65165ec66065f4af96f7b4ce53f8edac10775e0d82660aa98ca62125699f7809dac8cf1fc8d44a09cc44f0d04ee318fb0015e5d7dcd7a23f6a5d3b1dbbdf8aab207245edf079d71c6ef5b3fc04416L"
	c1 = long(c1, 16)

	#Encrypted message 2
	c2 = "0x1348effb7ff42372122f372020b9b22c8e053e048c72258ba7a2606c82129d1688ae6e0df7d4fb97b1009e7a3215aca9089a4dfd6e81351d81b3f4e1b358504f024892302cd72f51000f1664b2de9578fbb284427b04ef0a38135751864541515eada61b4c72e57382cf901922094b3fe0b5ebbdbac16dc572c392f6c9fbd01eL"
	c2 = long(c2, 16)

	x = modinv(37, N)

	a = (52*x)%N
	b = ((52**2) - 52*(37**2)*x)%N
	y = (a * (c2 - (a**3) * c1 + 2 * (b**3)))%N

	y = modinv(y, N)
	m1 = ((b * (c2 + 2 * (a**3) * c1 - (b**3))) *y )%N

	print "M1 = " + str(m1)
	t = (x*(m1 - (37**2)))%N
	print "t = " + str(hex(t))

```

Running the above script returns the following output

	0x576f772120596f757220666c61672069733a206469645f796f755f6b6e6f775f796f755f63616e5f736f6d6574696d65735f6763645f6f7574736964655f615f6575636c696465616e5f646f6d61696e0a

that converted to ASCII gives us the flag:

	Wow! Your flag is: did_you_know_you_can_sometimes_gcd_outside_a_euclidean_domain
