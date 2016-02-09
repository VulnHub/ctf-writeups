### Solved by Swappage

Web50 was a php web application challenge based on file inclusion (not really) and some code obfuscation.

the index.php page was accepting the GET parameter *p*, and by looking at it it was fairly obvious that it was dynamically loading pages, it wasn't a real inclusion tho; when sending the parameter

    p=index.php

the result was for the php source code to be displayed as part of the html code.

The result of the first iteration was the source of the index page to be leaked from the server

```php
<?php
// NULL Code Obfuscator
// www.null-life.com
include 'encoder.php';

error_reporting(0);
$code =
'GmjTplJrnv4Hd9uqLEm22igpg6kuJ9quCATTrlMu1/4SaZauTi7X0TRLp9VUftTTSASOrhZigOtTdfmuUy7TqgNvlOtTM9OpA2+U6wAhm+Eea936A2LUtXlz+YQaaNOmAHqB/hx926oDb5TrXy7UoF0p2q5SM86uFW+f/RYn0/V5LtOuUyqD7xRr07NTKYPvFGuAoRthnutdeoPiVDX583kE1+gaYpauTi7RoFwqg+8Ua9G1eQSa6FMmlecfa6zrC2eA+gAm1+gaYpanWi6IhFMu064WbZvhU2ia4hZRlOsHUZDhHXqW4Ad926xdIdf+EmmWrFo1+fNTa5/9Fi6IhFMu064WbZvhU2ia4hZRlOsHUZDhHXqW4Ad926ldIYPvFGuAoRthnutdeoPiVCfIhA4=';

$base = "\x62\x61\x73\x65\x36\x34\x5f\x64\x65\x63\x6f\x64\x65";
eval(NULLphp\getcode(basename(__FILE__), $base($code)));
?>
```
Since encoding is in place, i used the same technique to download the encoder.php to my local system and used a dirty trick to decode the obfuscated code.

```php
<?php

namespace NULLphp;

$seed = 13;
function rand() {
    global $seed;

    return ($seed = ($seed * 127 + 257) % 256);
}

function srand($init) {
    global $seed;

    $seed = $init;
}

function generateseed($string) {
    $output = 0;

    for ($i = 0; $i < strlen($string); $i++) {
        $output += ord($string[$i]);
    }

    return $output;
}

function getcode($filename, $code) {
    srand(generateseed($filename));

    $result = '';
    for ($i = 0; $i < strlen($code); $i++) {
        $result .= chr(ord($code[$i]) ^ rand());
    }

    return $result;
}

?>

```

and used a dirty trick to decode the obfuscated code, like replacing the *eval()* with *echo* :)

The result is the following decoded index page

```php
<?php
if (!empty($_GET['p'])) {
  $page = $_GET['p'];
}
else {
  $page = 'pages/home.tpl';
}

if (strpos($page, '..') !== false) {
  $page = 'pages/home.tpl';
}

$file = "./$page";

if (file_exists($file)) {
  echo file_get_contents("./$page");
}
else {
  echo file_get_contents('./pages/home.tpl');
}

?>
```

Now everything is clear, it's not an inclusion it's just a *file_get_content()* which actually displays the source code as output, with some restrictions to lock the user inside the webroot.

Let's try to download the login script then and see what happens. (let's get stright to the decoded one)

```php
<?php
if (!empty($_POST['email']) && !empty($_POST['password'])) {
  $email = $_POST['email'];
  $pass = $_POST['password'];
  // I can not disclose the real key at this moment
  // $key = 'random_php_obfuscation'; $key = '';
  if ($email === 'admin@hackermarket.onion' && $pass === 'admin') {
    echo 'well done! EKO{' . $key . '}';
  }
  else {
    echo 'Oh snap! Wrong credentials';
  }
}
else {
  header('Location: index.php');
}

?>
```

hey look, the flag is in the comments!

the flag was **EKO{random_php_obfuscation}**

