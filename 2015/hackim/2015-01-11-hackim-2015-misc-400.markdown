### Solved by superkojiman

A 400 point question with the following description:

> Find out the hidden gift in this SERVER : PORT - 54.165.191.231:2910 

Connecting to the target returns some data that appears to be garbage at first, but is actually in EBCDIC. Piping the output into "xxd -E" returns it in a readable format. Eg:

```
Well,  Try decoding this for me 

Decode this  e8ab2debf519c1b3dce054441840b844 : 
Enter decoded message:
```

Note that the returned hash actually changes after some time. Entering this hash in [crackstation.net](http://www.crackstation.net) returns "4 SEET". This string needs to be sent back to the server in EBCDIC as well. If we got it right, we get another hash and we have to do the procedure over again. After the second time, we get the flag. To simplify things, I wrote a dirty python script to exploit it:


```python
from socket import *
import codecs

s = socket(AF_INET, SOCK_STREAM)
s.connect(("54.165.191.231", 2910))

print s.recv(1024).decode('EBCDIC-CP-BE')
print s.recv(1024).decode('EBCDIC-CP-BE')
print "Enter decoded message:"
x = raw_input()
s.send(x.encode('EBCDIC-CP-BE'))

print s.recv(1024).decode('EBCDIC-CP-BE')
print s.recv(1024).decode('EBCDIC-CP-BE')
print s.recv(1024).decode('EBCDIC-CP-BE')

print "Enter decoded message:"
x = raw_input()
s.send(x.encode('EBCDIC-CP-BE'))

print s.recv(1024).decode('EBCDIC-CP-BE')
print s.recv(1024).decode('EBCDIC-CP-BE')
print s.recv(1024).decode('EBCDIC-CP-BE')

s.close()
```

And here it is in action: 


```text
# ./sploit.py 
Well,  Try decoding this for me 

Decode this  e8ab2debf519c1b3dce054441840b844 : 
Enter decoded message:
4 SEET
Good job ! 
Well,  Try decoding this for me 

Decode this  e1b9c9f26a2683990d01b6b7ca4acb68 : 
Enter decoded message:
Loftesness
Good job ! 
Finally you deserve this gift : flag{c0ding_enc0ding_d3c0ding}
```

The flag is: **flag{c0ding_enc0ding_d3c0ding}**

