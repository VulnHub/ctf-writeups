AlexCTF forensics2
---

### Solved by barrebas


Forensics, not something I usually do! We're handed a core dump file. It belongs to the mutt program, a mail client. The idea is to log in to the remote mailbox to grab the flag. 

```strings``` is a hacker's best friend. I quickly stumble upon this:
```bash
tp_pass = "e. en kv,dvlejhgouehg;oueh fenjhqeouhfouehejbge ef"
set from = "alexctf@example.com"
set realname = "alexctf"
set folder = "imaps://imap.example.com:993"
set spoolfile = "+INBOX"
```

But that ```tp_pass``` (probably *smtp_pass*) is not the correct password. The mutt documentation suggests that the password could be encrypted on disk. The core dump contains the same string twice. 
```
]/Drafts
 Old:%o!
mple.com
dksgkpdjg;kdj;gkje;gj;dkgv a enpginewognvln owkge  noejne
e. en kv,dvlejhgouehg;oueh fenjhqeouhfouehejbge ef
gpg --no-verbose --export --armor %r
phrA
```

On a hunch, I decided to try ```dksgkpdjg;kdj;gkje;gj;dkgv a enpginewognvln owkge  noejne``` as the password and this worked!

```bash
user@lithium:~/alexctf/fore2$ nc 195.154.53.62 2222

Email: alexctf@example.com
Password: dksgkpdjg;kdj;gkje;gj;dkgv a enpginewognvln owkge  noejne
1 new unread flag
ALEXCTF{Mu77_Th3_CoRe}
```

