---
layout: post
title: "PoliCTF 2015 Exorcise"
date: 2015-07-12 17:18:01 -0400
author: [barrebas]
comments: true
categories: [polictf]
---

### Solved by barrebas

Exorcise was a 50 point crypto challenge for PoliCTF.

<!--more-->

We're asked to connect to `exorcise.polictf.it:80`. Upon connecting, we're presented with a hexadecimal string. I pressed return and got another:

```bash
bas@tritonal:~$ nc exorcise.polictf.it 80
2e0540472c37151c4e007f481c2a0110311204084f

2e0541495b161248101c2a11122d16102d1608091902027f0d071c2c53050a061f05380d410f0a2a531f1e1907053d3310543e5d1c3a512653020c09461809025b341111475310451b3a014736000c4d0404002c1c4f142d164805001f107f094114103110074c190344283a00063b110c26413a00
```

Because of the challenge title, I xor'ed the second string using the first as a key and got gibberish. I decided to reconnect and send a bunch of NULL bytes (Ctrl+Space). When I xor'ed that string against the first, I got this:

```
What's your name^i%UX`C=h=pN^xPdaW]j,1ISJC't yru4OBwguHi! What's your name^G
```

Hmmmmm. I reconnected again, sent a bunch of A characters and the resulting string was xor'ed vs 0x41:

```python
e = "2e0541263a1e352928321e70321e32711e32282c312d241e382e341e32292e342d251e292037241e322e2d3724251e1e28351e702f1e741e3224223c272d20263a1e352928321e70321e32711e32282c312d241e382e341e32292e342d251e292037241e322e2d3724251e1e28351e702f1e741e3224223c272d2026557f3d0e490a3044533e01557f010c0c14050b38591b1d36004802101f173e0f04561c30064f1c040a063e3d050d7f50022d503b0a450412124c150f1e7f1f0d105d7f7e07371642281a1a0850180d3a0a0a553e010d4f1f19172b480056072a1d0c04560a027f280c1d2d554e3d4c2b1616"
msg = [e[start:start+2] for start in xrange(0, len(e), 2)]

out = ""
for i in range(len(msg)):
    b = (int(msg[i], 16))
    out += chr(0x41^b)
print out
```

```bash
bas@tritonal:~/tmp/polictf/exorcise$ python exorcise.py
oDg{_this_1s_s0_simple_you_should_have_solved__it_1n_5_sec}flag{_this_1s_s0_simple_you_should_have_solved__it_1n_5_sec}flag
```

There's the flag: `flag{_this_1s_s0_simple_you_should_have_solved__it_1n_5_sec}`. Took me a bit longer than 5 seconds...

