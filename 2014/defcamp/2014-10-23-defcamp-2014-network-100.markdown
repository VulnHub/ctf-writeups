### Solved by barrebas

For this challenge, we're given the following clue:

```
My manager lives at 10.13.37.21. Any guest is always welcome. But he has a secret. Can you find it out?
```

I did a simple nmap scan of that box:

```bash
Starting Nmap 6.00 ( http://nmap.org ) at 2014-10-18 13:08 CEST
Nmap scan report for 10.13.37.21
Host is up (0.024s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

The webpage has a login form, but I didn't have any credentials. The clue did say guests are always welcome, but ```guest:guest``` didn't work. However, these credentials *did* work for ```ssh```! So I landed a shell on the box:

```bash
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-37-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Sat Oct 18 00:05:47 2014 from 10.13.37.1
guest@n1:~$ ls
toolkit
guest@n1:~$ ls -alh
total 32K
drwxr-xr-x 4 guest guest 4.0K Oct 18 00:06 .
drwxr-xr-x 4 root  root  4.0K Oct 17 01:40 ..
-rw------- 1 guest guest    0 Oct 18 00:06 .bash_history
-rw-r--r-- 1 guest guest  220 Oct 17 01:40 .bash_logout
-rw-r--r-- 1 guest guest 3.6K Oct 17 01:40 .bashrc
drwx------ 2 guest guest 4.0K Oct 17 01:42 .cache
-rw-r--r-- 1 guest guest  675 Oct 17 01:40 .profile
drwxr-xr-t 2 root  root  4.0K Oct 17 01:41 toolkit
-rw------- 1 guest guest  777 Oct 18 00:06 .viminfo
```

For some reason, we could use `tcpdump` to look at incoming traffic:

```bash
guest@n1:~/toolkit$ ./tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
11:13:03.921683 IP n1.ssh > 10.13.37.1.52543: Flags [P.], seq 1154997930:1154998122, ack 2969808471, win 349, options [nop,nop,TS val 10598245 ecr 188495], length 192
11:13:03.934082 IP 10.13.37.1.52543 > n1.ssh: Flags [.], ack 192, win 661, options [nop,nop,TS val 188517 ecr 10598245], length 0
11:13:03.975864 IP 10.13.37.1.37658 > n1.41730: Flags [S], seq 2811437647, win 1024, options [mss 1369], length 0
11:13:03.975882 IP n1.41730 > 10.13.37.1.37658: Flags [R.], seq 0, ack 2811437648, win 0, length 0
11:13:03.987686 IP 10.13.37.1.43518 > n1.41511: Flags [S], seq 4021855713, win 1024, options [mss 1369], length 0
11:13:03.987704 IP n1.41511 > 10.13.37.1.43518: Flags [R.], seq 0, ack 402185571
```

But we're looking for a flag! I wanted to see what made the login page tick.

```bash
guest@n1:~/toolkit$ cat /var/www/html/index.php
<?php

error_reporting(0);
ini_set('display_errors', 0);

if(isset($_POST['pass']) && strlen($_POST['pass']) < 12) {
    echo 'Validation failed: The password field too short.'; die();
}

if(isset($_POST['user'])) {
  if($_POST['user'] == 'manager' && md5($_POST['pass']) == 'e701a78ce9d38201e9fc17737be0996d') {
    echo 'The secret is behind <strong>0f388689dc4728cfde0de9a1ee47c8d3</strong>. Don\'t tell anyone!';
  } else {
    echo 'Wrong username or password.';
  }
  die();
}

?>
<!DOCTYPE html>
...snip...
```

Apparently, the flag is 'behind' `0f388689dc4728cfde0de9a1ee47c8d3`. This md5 hash decrypts to `ididyourmom`, which is also the flag. What a nasty secret this manager has!
