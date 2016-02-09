---
layout: post
title: "Backdoor CTF 2015 LHC"
date: 2015-04-02 16:16:05 -0400
author: [barrebas]
comments: true
categories: [backdoor]
---

### Solved by barrebas

`LHC` was a short & sweet challenge in Backdoor CTF. It had a nice 'aha-Erlebnis' moment.

During the CTF, the organizers dropped this challenge. The description mentioned that the flag was hidden in a data file. This data file was kindly provided by the Large Hadron Collider and was 2049 GB large.

I'll let that sink in.

Two-thousand and forty nine **gigabytes**. Downloading it would take more than twenty days. The flag was in the middle of the file, but that still meant downloading more than a terabyte of data.

I fired up `curl` and looked at the download:

```
bas@tritonal:~/tmp/bckdr/medusa$ curl -vvv https://lhc-cdn.herokuapp.com/data.txt > /dev/null
* About to connect() to lhc-cdn.herokuapp.com port 443 (#0)
*   Trying 107.21.223.88...
...snip...
> GET /data.txt HTTP/1.1
> User-Agent: curl/7.26.0
> Host: lhc-cdn.herokuapp.com
> Accept: */*
> 
* additional stuff not fine transfer.c:1037: 0 0
* HTTP 1.1 or later with persistent connection, pipelining supported
< HTTP/1.1 200 OK
< Server: Cowboy
< Connection: keep-alive
< X-Powered-By: Express
< Accept-Ranges: bytes
< Content-Type: text/plain; charset=utf-8
< Content-Length: 2200000000000
< Date: Wed, 01 Apr 2015 21:44:48 GMT
< Via: 1.1 vegur
```

Yup, 2200000000000 bytes of data. But wait! `curl` has the option to resume a broken download; that meant that I could control where the download would start. I issued this command and started searching the output for the flag:

```bash
$ curl -vvv https://lhc-cdn.herokuapp.com/data.txt -C 1100000000000 > lhc-middle
...snip...
$ strings -n 20 ./lhc-middle
```

This gave me part of the flag; the description said it was in the middle of the datafile, so I subtracted another 1000 bytes:

```
$ curl -vvv https://lhc-cdn.herokuapp.com/data.txt -C 1099999998999 > lhc-middle
$ strings lhc-middle -n 20
              The flag is: [REDACTED]
```

Simple, really, once you know the trick.

