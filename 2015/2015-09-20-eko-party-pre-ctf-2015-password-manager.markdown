---
layout: post
title: "Eko Party Pre-CTF 2015 Password Manager"
date: 2015-09-20 23:33:36 -0400
author: [NullMode]
comments: true
categories: [ekoparty]
---

### Solved by NullMode

Password Safe 3 file. Got the hash using John's pwsafe2john:

```
mypasswords:$pwsafe$*3*1f295812c442fa6614c8f5442b701be81a817fef1a05e095a420666eabc37e9b*2048*a482b17009d206b7fa3495b40e4113a296efc23bd92b68905929277bdae4a246
```

Need to crack it. 

NullMode's got this!

Cracked with maskprocessor piped into john

based on the hint [a-zA-Z0-9]{0,4} we can come up with a very simple mask attack using mask processor piped into john

```
mp64.bin -1 "?d?l?u" -i "?1?1?1?1" | ./john --stdin pwsafe.txt
```

mask processor, defined a custom mask which included lowercase [a-z], uppercase [A-Z], and digits [0-9], incremental for {0,4} with -i and the mask defined to a length of 4 ?1's.

```
mp64.bin -1 "?d?l?u" -i "?1?1?1?1" | ./john --stdin pwsafe.txt
Loaded 1 password hash (pwsafe, Password Safe [SHA256 32/64])
Will run 4 OpenMP threads
Press Ctrl-C to abort, or send SIGUSR1 to john process for status
 Ek0              (mypasswords)
1g 0:00:00:31  0.03134g/s 4981p/s 4981c/s 4981C/s EjY..Ek1
Use the "--show" option to display all of the cracked passwords reliably
Session completed

mypasswords:Ek0
```

Installed password safe on a windows 7 vm (took a snapshot of course!)

opened the file with the password above and there's an entry with the flag in it :)

![](/images/2015/ekoparty/passwordmanager/flag.png)

**EKO{password_managers_rulez}**

