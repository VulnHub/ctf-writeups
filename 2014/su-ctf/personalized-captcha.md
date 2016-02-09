### Solved by Swappage

Second CTF for the VulnHub team, and lots of fun with these puzzles.

This writeup is all about an interesting forensics and web game named "Personalized captcha" where the players were challenged to discover the value of a captcha string by analizing a provided pcap file.

<!-- more -->

The pcap file was ~9MB in size, which is not big, but for sure bigger then usual for a CTF puzzle, and by a first look at it using wireshark it looked fairly messy, especially in terms of HTTP traffic, considering that a fairly good amount of HTTP requests and streams where in there.

Sorting things out by hand would have been fairly challenging, and when playing a CTF, you really have to be as fast as possible, so i decided to rely on a NFAT (Network forensics analysis tool) named Xplico, available in backtrack for dissecting it.

This tool is excellent for dissecting even big (100MB+) pcap files, and has really powerful filtering and search features, especially when you need to rebuild content for analysis, like web pages, images, audio files, voip communications and so on.

![](/images/2014/sharif/captcha/packet_summary.png)

Punching the pcap into the software made things clearer and more understandable then by simply looking at raw packets in wireshark, and in the first place i started looking at the sites the user visited.

The challenge hinted the players about the domain *captcha.ctf.sharif.edu* being down, which of course dragged me into digging more in depth about anything that i could find in the pcap about that domain.

![](/images/2014/sharif/captcha/sharif_captcha.png)

A quick search revealed that the user for whom we were trying to rebuild the captcha actually visited that site; i focused some effort in dissecting the communications between the user and the server, i went sure on a post request that seemed interesting but unfortunately the user never really submitted the captcha, so, unfortunately, not a chance to grab it from the POST request, even if it wasn't encrypted.

![](/images/2014/sharif/captcha/post.png)

Well, in the end this was a 300 points worth puzzle, wouldn't it be too easy that way?

Anyway, back to the challenge, at this point it was probably a good idea to try to rebuild the pages content.

Xplico in these things really rocks the world, because it can preview the content of web pages by rebuilding them entirely off a pcap file, i tried previewing the page as i usually do, but this time something went wrong.

![](/images/2014/sharif/captcha/badpage.png)

For the good actually, because this helped me in understanding the point of the puzzle (more on this later), but the page looked really weird, by quickly inspecting the source, I noticed that 2 CSS files were included in the html, but if we look at them in Xplico we can notice that the *style.css* was 0 bytes in size.

```html
<!doctype html>
<html>
<head>
        <meta charset="UTF-8">
        <title>The Captcha</title>
        <link rel="stylesheet" href="css/bootstrap.css">
        <link rel="stylesheet" href="css/style.css">
</head>
<body>
...
</body>
</html>
```

![](/images/2014/sharif/captcha/wrongsize.png)

mhh.. odd isn't it? i double checked using wireshark and it actually resulted that something DID went wrong in pcap dissection by Xplico, did the challenge author know something i didn't? was this done on purpose?

Yet, wireshark revealed the truth and at this point i decided to export the files directly from wireshark and restore their original paths so that the page would display properly; luckly wireshark provides an awesome feature to export files from HTTP strams, so it was just a matter of a couple of mouse clicks to get everything i needed.

Opening the web page in the browser revealed what was the page as it appeared to the user whom this traffic capture belonged to

![](/images/2014/sharif/captcha/wholepage.png)

except for the fact, that the captcha was empty.

Yet, pieces of the puzzle were starting to make sense if put togeder; now i had a captcha field, but where are the links that i could see without the *style.css*?

Let's take a look at the whole page code

```html
<!doctype html>
<html>
<head>
        <meta charset="UTF-8">
        <title>The Captcha</title>
        <link rel="stylesheet" href="css/bootstrap.css">
        <link rel="stylesheet" href="css/style.css">
</head>
<body>
<div class="container">
        <div class="row">
                <div class="col-xs-8 col-xs-offset-2 col-md-6 col-md-offset-3">
                        <form method="post">
                                <textarea class="form-control mb5" placeholder="comment" name="comment"></textarea>
                                <input type="text" class="form-control mb5" placeholder="captcha" name="captcha">

                                <div id="captcha">
                                        <a href="http://en.wikipedia.org/wiki/Hack">P</a>
                                        <a href="http://wordpress.org/plugins/captcha/">E</a>
                                        <a href="http://wordpress.org/mobile/">A</a>
                                        <a href="http://captchas.net/">C</a>
                                        <a href="http://en.wikipedia.org/wiki/Hack">E</a>

                                        <a href="http://www.thefreedictionary.com/hack">Y</a>
                                        <a href="http://www.php.net/manual/en/security.database.sql-injection.php">E</a>
                                        <a href="http://www.unixwiz.net/techtips/sql-injection.html">T</a>

                                        <a href="http://www.google.com/recaptcha/intro/index.html">V</a>
                                        <a href="http://en.wikipedia.org/wiki/OWASP">U</a>
                                        <a href="http://www.phpcaptcha.org/documentation/quickstart-guide/">L</a>

                                        <a href="http://www.merriam-webster.com/dictionary/hack">A</a>
                                        <a href="http://www.urbandictionary.com/define.php?term=hack">N</a>
                                        <a href="http://www.hackthissite.org/">D</a>

                                        <a href="http://www.phpcaptcha.org/posts/wordpress-plugin-released/">A</a>

                                        <a href="http://en.wiktionary.org/wiki/hack">H</a>
                                        <a href="http://www.captcha.net/">A</a>
                                        <a href="http://captchas.net/sample/php/">C</a>
                                        <a href="http://www.phpcaptcha.org/download/wordpress-plugin/">K</a>
                                </div>
                                <button type="submit" class="btn btn-primary btn-block">send</button>
                        </form>
                </div>
        </div>
</div>
</body>
</html>
```
and at the *style.css*

```css
#captcha {
  width: 300px;
  height: 100px;
  margin-right: auto;
  margin-left: auto;
  position: relative;
  background-color: #000000;
  white-space-collapse: discard;
}
#captcha a {
  font-size: 15px;
  pointer-events: none;
  cursor: default;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
  position: absolute;
  display: inline-block;
}
#captcha a,
#captcha a:hover,
#captcha a:focus {
  color: #000000;
}
#captcha a:visited {
  color: #ff0000;
}
#captcha a:nth-child(1) {
  left: 3.45699886px;
  top: 26.46903004px;
  -webkit-transform: rotate(-3.45879735deg);
  -ms-transform: rotate(-3.45879735deg);
  transform: rotate(-3.45879735deg);
}
#captcha a:nth-child(2) {
  left: 19.42964103px;
  top: 28.76705699px;
  -webkit-transform: rotate(-9.34001927deg);
  -ms-transform: rotate(-9.34001927deg);
  transform: rotate(-9.34001927deg);
}
```

Oh... Ok, now i get it, the links are building the captcha!

Let's try this out: i tried to visit one of the links from the page source, so that they would appear as visited to the browser, and by refreshing the captcha page i was presented with a pleasant surprise.

![](/images/2014/sharif/captcha/partial_captcha.png)

So now i had the point of the challenge! Basically the captcha string depended on the sites visited by the user, what i needed to do, to obtain the captcha string was to verify which sites among the ones in the captcha page were present in the pcap, respecting the following criteria:

- the site should have been visited before http://captcha.ctf.sharif.edu/captcha/ was visited (there were a couple of browsing sessions after that)
- the source ip address should have been the same (better making sure not to include other potential users)
- the browser used should have been always the same as the one used for visiting the captcha page (in the pcap multiple user agents were present)

doing the search by end in wireshark would have been frustrating, even by using filters, so i created a filter like the following

```text
frame.number < 14127 && ip.src == 10.10.12.30 && http.user_agent == "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/536.11 (KHTML, like Gecko) Ubuntu/12.04 Chromium/20.0.1132.47 Chrome/20.0.1132.47 Safari/536.11" && http.request.full_uri == ""
```

And with the help of some basic bash scripting i cycled through the links in the captcha page to see if they were visited or not.

```bash
#!/bin/bash

URLS=$(grep "a href" captcha.htm | awk -F '\"' '{print $2}')

for url in $URLS; do
    FILTER="frame.number < 14127 && ip.src == 10.10.12.30 && http.user_agent == \"Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/536.11 (KHTML, like Gecko) Ubuntu/12.04 Chromium/20.0.1132.47 Chrome/20.0.1132.47 Safari/536.11\" && http.request.full_uri == \"$url\""
    VISITED=$(tshark -R "$FILTER" -r captcha.pcap 2> /dev/null | wc -l)
    if [ $VISITED -ne 0 ]; then
                    echo $url
    fi
done
```

All was left to do was to take the returned URLs, open them in the browser and see the resulting captcha

![](/images/2014/sharif/captcha/solvedcaptcha.png)

What else can i say? well, i think i'll probably submit this pcap file to the Xplico dev team so that they can check why *style.css* wasn't decoded properly.
