---
layout: post
title: "0CTF 2015 Quals Forward"
date: 2015-04-03 23:22:22 -0400
author: [swappage]
comments: true
categories: [0ctf]
---

### Solved by Swappage

Forward was fun, a nice web challenge involving some lateral thinking and php source analysis.

We were provided with the following PHP source code, and we were asked to retrieve the flag.

```php
<?php
   if (isset($_GET['view-source'])) {
       show_source(__FILE__);
       exit();
   }
   include("./inc.php"); // key & database config

   function err($str){ die("<script>alert(\"$str\");window.location.href='./';</script>"); }

   $nonce = mt_rand();

   extract($_GET); // this is my backdoor :)

   if (empty($_POST['key'])) {

       err("Parameter Missing!");
   }

   if ($_POST['key'] !== $key) {
       err("You Are Not Authorized!");
   }

   $conn = mysql_connect($host, $user, $pass);

   if (!$conn) {
       err("Database Error, Please Contact with GameMaster!");
   }

   $query = isset($_POST['query']) ? bin2hex($_POST['query']) : "SELECT flag FROM forward.flag";
   $res = mysql_query($query);
   if (FALSE == $res) {
       err("Database Error, Please Contact with GameMaster!");
   }

   $row = mysql_fetch_array($res);

   if ($debug) {
       echo "HOST:\t{$host}<br/>";
       echo "USER:\t{$user}<br/>";
   }

   echo "<del>FLAG:\t0ctf{</del>" . sha1($nonce . md5($row['flag'])) . "<del>}</del><br/>"; // not real flag

   mysql_close($conn);

```

The source was also giving out some free hints on what to look for to solve the challenge

what we can immediately spot is a non sanitized

```php
    extract($_GET);
```

that we can exploit to overwrite variables via GET requests.

*extract()* in PHP opens up the applications to attacks that involves variables redefinition: what we can do here is reassign all the variables that were defined before *extract()* is called, and therefore alter the application behavior.

The rest of the application, basically reads the flag from the database, appends it to the random nonce and performs a *sha1(md5())*

## Exploitation

So the flag is read from the database, but there is no way we can have it returned from the web application itself, because even if we can easily bypass the authorization key check by reassigning *$key*, from the query to the output, the flag value is tampered in a non reversible way.

Our only chance is to intercept it in the middle.

By enabling the debug

    ?key=swappage&debug=1 // get request

we are returned the values of the username and host variables as defined in *./inc.php*

And we can notice that the MySQL database is running on a public IP, and by a quick check, we can also confirm that port 3306 is open to access from everyone.

What we can try to do then is to abuse *extract()* to overwrite

- $key: to bypass the authentication check
- $debug: to display some useful variable values
- $host: to force the application to connect to an arbitrary host on port 3306 instead of the original MySQL Server.

If the application can be forced to connect to an arbitrary host, a forwarding (hint hint from the challenge name) can be set up to perform a MITM attack on the database communication and intercept the flag on the wire.

Let's set up socat and give it a try

    $ socat TCP-LISTEN:3306,fork TCP:202.112.28.121:3306

i sent the following request to the web server

{% img center /images/2015/0ctf/forward/burp.png %}

and the webserver connected to me as a stepping stone for connecting to the backend database, giving me a chance to sniff all the traffic and read the flag in clear text

{% img center /images/2015/0ctf/forward/flag.png %}

The flag is

    0ctf{w3ll_d0ne_guY}

