###Solved by bitvijays

Can you find the flag?


irc.pcap

We were provided with a pcap with a IRC conversation in plaintext and had to find the flag. This was pretty easy.
A part of the conversation is below: If we look carefully, the flag is sent in multiple messages

```
***SNIP***
PING irc.capturetheflag.withgoogle.com
:irc.capturetheflag.withgoogle.com PONG irc.capturetheflag.withgoogle.com :irc.capturetheflag.withgoogle.com
PRIVMSG #ctf :but it's plaintext, so I guess 5 eyes will get this flag before anyone else
PRIVMSG #ctf :so let's do it as a group
:andrewg!~poppopret@agriffiths.c.gctf-2015-admins.google.com.internal PRIVMSG #ctf :CTF{
PING irc.capturetheflag.withgoogle.com
:irc.capturetheflag.withgoogle.com PONG irc.capturetheflag.withgoogle.com :irc.capturetheflag.withgoogle.com
:itsl0wk3y!~poppopret@itsl0wk3y.c.gctf-2015-admins.google.com.internal PRIVMSG #ctf :some_
PRIVMSG #ctf :leaks_
:andrewg!~poppopret@agriffiths.c.gctf-2015-admins.google.com.internal PRIVMSG #ctf :are_
PRIVMSG #ctf :good_
:itsl0wk3y!~poppopret@itsl0wk3y.c.gctf-2015-admins.google.com.internal PRIVMSG #ctf :leaks_
:andrewg!~poppopret@agriffiths.c.gctf-2015-admins.google.com.internal PRIVMSG #ctf :}
PING irc.capturetheflag.withgoogle.com
:irc.capturetheflag.withgoogle.com PONG
***SNIP***

The flag is ***CTF{some_leaks_are_good_leaks_}***
