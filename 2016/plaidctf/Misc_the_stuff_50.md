###Solved by bitvijays

the stuff
Misc (50 pts)
Can you believe Ryan uses Bing?

We were provided by a pcap file which contained multiple traffic tcp,http, smtp, dns etc.

Focussing on SMTP, we found a Email transfer of password protected zip file
```
--=-zAAY+FBv9yZgwoZy4KHy
Content-Type: text/plain
Content-Transfer-Encoding: 7bit

Yo, I got the stuff.

--=-zAAY+FBv9yZgwoZy4KHy
Content-Type: application/zip; name="flag.zip"
Content-Disposition: attachment; filename="flag.zip"
Content-Transfer-Encoding: base64
```

Extracting the zip file, we required a password as expected. To be frank, I did not saw another smtp stream, so used strings and grepped for pass
```
cat st | sort | uniq | grep pass
loginnet.passport.com
login.passport.com
msnialogin.passport.com
nexus.passport.com
 pst.microsoftpassportsupport.net
Yo, you'll need this too: super_password1
```

Otherwise, checking the second smtp stream we get
```
Message-ID: <1460851191.7821.2.camel@ubuntu>
Subject: Wait, hang on
From: John Doe <jdoe@example.com>
To: jsmith@example.com
Date: Sat, 16 Apr 2016 16:59:51 -0700
Content-Type: text/plain
X-Mailer: Evolution 3.10.4-0ubuntu2 
Mime-Version: 1.0

Yo, you'll need this too: super_password1
```

Using this password to extract the flag from the zip file.

The flag is **PCTF{LOOK_MA_NO_SSL}**
