# DefCon416 CTF

This was the first CTF hosted by Toronto's DefCon chapter. We were pointed to a machine and had to find all the different flags scattered about. This was pretty fast paced, and unfortunately we didn't get to document or screenshot everything along the way. Therefore this writeup is going to be more of a quick summary of how we got some of the flags, or at least the ones we can remember. 

## Flag #1

The first target was `http://galahad.dc416.com`. The source code of `/staff` (found with Burp Spider & dirbuster) revealed a JavaScript file `ob.js`, that contained the following:

```
var _0x4c9e=["\x44\x43\x34\x31\x36","\x73\x79\x6E\x74\x31\x7B\x7A\x30\x30\x61\x70\x34\x78\x72\x7D","\x4E\x6F\x6E\x65","\x70\x61\x72\x61\x74\x75\x3A","\x6C\x6F\x67"];var CTF=_0x4c9e[0];if(CTF== _0x4c9e[0]){FLAG= _0x4c9e[1]}else {FLAG= _0x4c9e[2]};console[_0x4c9e[4]](_0x4c9e[3]);console[_0x4c9e[4]](FLAG)
```

This returned `synt1{z00ap4xr}`, which was ROT-13 encoded. Decoding it returned `flag1{m00nc4ke}`.

## Flag #4

A nikto scan on the server revealed the `/admin` directory on `http://galahad.dc416.com`, which contained a link to `enc.zip`. This in turn contained a file called `enc.pyc`. Using `uncomplyer`, we were able to decompile it into the following:

```
# uncompyler.py ../enc.pyc 
# 2016.09.10 12:50:14 EDT
DESC = 'C4N YOU 1D3N71FY 7H3 FL46?'
str1 = 'FLAG4{'
str2 = '______'
str3 = '0'
str4 = '_____________________'
str5 = '__________________'
str6 = '____'
str7 = '1'
str8 = '_______'
str9 = '1'
str10 = '____________________'
str11 = '__________________________'
str12 = '}'
```

We had tackled a similar challenge before in [th3jackers 2015 CTF](https://github.com/VulnHub/ctf-writeups/blob/05ceec63c98bb42348462d3ba384d521db3654be/2015/th3jackers/misc100.md). Basically, the number of underscores corresponded to a letter in the alphabet. This gave us the fourth flag: `FLAG4{f0urd1g1tz}`.

## Flag #5

The fifth flag had to be downloaded from the FTP service on `galahad.dc416.com`. The clue was actually on port 50000, which had an NSA ASCII banner and text that changed at the bottom.

```
NNNNNNNN        NNNNNNNN   SSSSSSSSSSSSSSS              AAA
N:::::::N       N::::::N SS:::::::::::::::S            N:::A
S::::::::N      E::::::NC:::::SUSRSI::::::S           T:::::Y
N:::::::::N     N::::::NS:::::S     SSSSSSS          A:::::::A
N::::::::::N    N::::::NS:::::S                     A:::::::::A
N:::::::::::N   N::::::NS:::::S                    A:::::A:::::A
T:::::::H::::N  R::::::O U::::SSSS                G:::::A H:::::A
N::::::N N::::N N::::::N  SS::::::SSSSS          A:::::A   A:::::A
N::::::N  N::::N:::::::N    SSS::::::::SS       A:::::A     A:::::A
N::::::N   N:::::::::::N       SSOBSC::::S     A:::::AARAIAATY:::::A
3::::::N    4::::::::::N            3:::::S   4:::::::::::::::::::::A
N::::::N     N:::::::::N            S:::::S  A:::::AAAAAAAAAAAAA:::::A
3::::::N      4::::::::NS3SSSS4     S:::::S 0:::::d             0:::::a
N::::::N       N:::::::NS::::::SUDPSS:::::SA:::::A               A:::::A
N::::::N        N::::::NS:::::::::::::::SSA:::::A                 A:::::A
NNNNNNNN         NNNNNNN SSSSSSSSSSSSSSS AAAAAAA                   AAAAAAA


This is for staff only
```

One of the texts was the following number sequence:

```
31337 7331 31338 8331 ____
```

We noticed that some letters in the NSA banner didn't quite belong. Most of the letters were `N`, `S`, `A`, and `:`. However once we stripped those, we were left with the following hidden message:

```
# cat nsa | tr -d 'NSA:'
                         
                    
      ECURI           TY
                    
                         
                       
TH  RO U                G H
                 
                  
          OBC     RITY
3    4            3   4
                   
3      434      0d             0a
       UDP
```

Put together, it reads:

```
(S)ECURITY THROUGH OB(S)C(U)RITY        <-- dunno where the 'U' went :)
34343434 0d 0a UDP
```

We tried connecting to port 3434 until we realized that it was actually `0x34`, which was `4`. It suddenly made sense that the missing 4 digits in the number sequence was `4444`. We tried port knocking on UDP but no luck. A hint was later provided that gave out the correct sequence: `4444:udp 8331:tcp 7331:tcp 31337:tcp 31338:tcp`. We used `knockd` with this sequence, which opened FTP for us. There were two files called `bar.pcap` and `foo.pcap`.

The fifth flag was found in `bar.pcap`. We used `tcpflow` to break it apart and just grepped for `flag`:

```
# grep -i flag *
104.020.064.056.00080-159.203.035.128.44594:        <title>flag5{th3fuzz} - Pastebin.com</title>
104.020.064.056.00080-159.203.035.128.44594:        <meta property="og:title" content="flag5{th3fuzz} - Pastebin.com" />
.
.
.
``` 

## Flag #6

A clue to the sixth flag was found in the second pcap file, `foo.pcap`. We isolated the data stream that contained a JPEG, `power.jpg` and saved it as a raw file. We then used `foremost` to extract the jpeg, which gave us an image of a power plant with some co-ordinates on the top left, as well as a username, `nitro`. 

![](/images/2016/defcon416/power.jpg)

Armed with clues about power plants, NSA and Stuxnet, we queried Google and found reference to [Nitro Zeus](http://arstechnica.com/tech-policy/2016/02/massive-us-planned-cyberattack-against-iran-went-well-beyond-stuxnet/). We tried connecting to `galahad.dc416.com` with the crdentials `nitro:zeus` and were presented with a Tic-Tac-Toe game.

The game requires you to win 3/5 games. The starting player (you or computer) seems randomly decided.  By continuously disconnecting/reconnecting until you move first, it's pretty easy to engineer a 3/5 victory (unless you're good enough to win a game when you don't move first).  So the sequence went: `win, draw, win, draw, win`.

This gave us the flag and a new hint:

```
Flag: flag6{s1xfl4gs}
Hint: Did you know that Galahad is just one of a few round-table knights?
```

We pulled down a list of all the Knights of the Round Table and did a DNS query on each one, until we found `lancelot.dc416.com`.


## Flag #9

A `nikto` scan on `http://lancelot.dc416.com` found the `/webmail` directory. It seems this was supposed to be a SQL injection challenge, but we were able to solve it just by viewing the source file, and clicking on `query.php`.  This returned `adminflag9Ihopeyoudidntdothismanuallylol`.

`Flag: flag9{Ihopeyoudidntdothismanuallylol}`

Alternatively, `' or '1'='1' -- -` also worked.

## Flag #10

The SSH login credentials to `lancelot.dc416.com` were provided as a hint. Logging in, we found we were in some kind of Python jail. While brushing up on our Python jail escape online we came across this previous [CTF challenge](http://codezen.fr/2012/10/25/hack-lu-ctf-python-jail-writeup/), which seemed to be very similar to what we were faced with. It turned out we were able to read files (verified by reading `/etc/passwd`) using the same solution.

A bit of guess work led us to the flag:

```
5;s=raw_input();exec(s)
f = (t for t in (42).__class__.__base__.__subclasses__() if t.__name__ == 'file').next()('/home/michael/flag10')
Returned: 5
5;s=raw_input();exec(s)
a = f.read()
Returned: Congrats!, you are free!

        .-.
       (   )
        |~|
        | |
        | |       _.--._
        |~|~:'--~'      |
        | | :           |
        | | :     _.--._|
        |~|~`'--~'
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |
        | |   DC416
   _____|_|_________
 

here is your flag: flag10{unixgiants} 

For the last flag, there is no more jeopardy. anarchy mode: on

simply get root on this server and capture it. there is at least one (almost easy) way.

good luck
```

In the end, we were just pipped at the post for first, but managed to score second place! Some challenges really stumped us for a while, but it was always a good high to finally score that flag when we did. Fun times. Kudos to DEFCON Toronto on their [first CTF](https://twitter.com/defcon_toronto/status/774769112383950848).

![](/images/2016/defcon416/scoreboard.png)
