---
layout: post
title: "CSAW Quals 2015 Alexander Taylor"
date: 2015-09-23 19:17:29 -0400
author: [superkojiman, bitvijays]
comments: true
categories: [csaw]
---

### Solved by superkojiman and bitvijays

The recon challenge starts at [http://fuzyll.com/csaw2015/start](http://fuzyll.com/csaw2015/start)

The first clue:

```
CSAW 2015 FUZYLL RECON PART 1 OF ?: Oh, good, you can use HTTP! The next part is at /csaw2015/<the acronym for my university's hacking club>.
```

Look at fuzyll's [LinkedIn](https://www.linkedin.com/in/fuzyll) and found that he was president of the Whitehatter's Computer Security Club (wcsc). This leads us to:

[http://fuzyll.com/csaw2015/wcsc](http://fuzyll.com/csaw2015/wcsc)

This displays:

```
CSAW 2015 FUZYLL RECON PART 2 OF ?: TmljZSB3b3JrISBUaGUgbmV4dCBwYXJ0IGlzIGF0IC9jc2F3MjAxNS88bXkgc3VwZXIgc21hc2ggYnJvdGhlcnMgbWFpbj4uCg==
```

Base64 encoded string. Let's decode it to get:

```
Nice work! The next part is at /csaw2015/<my super smash brothers main>.
```

A quick google leads us to a YouTube link [https://www.youtube.com/watch?v=MbRKFWyPQkQ](https://www.youtube.com/watch?v=MbRKFWyPQkQ)

So it's yoshi: [http://fuzyll.com/csaw2015/yoshi](http://fuzyll.com/csaw2015/yoshi)

We get an image of Yoshi. Run exiftool on it and in the comments section we get:

```
Comment                         : CSAW 2015 FUZYLL RECON PART 3 OF ?: Isn't Yoshi the best?! The next egg in your hunt can be found at /csaw2015/<the cryptosystem I had to break in my first defcon qualifier>.
```

Searched for crypto systems for DefCon and bitvijays found [http://www.vnsecurity.net/ctf%20-%20clgt%20crew/2010/05/25/defcon-18-quals-writeups-collection.html](http://www.vnsecurity.net/ctf%20-%20clgt%20crew/2010/05/25/defcon-18-quals-writeups-collection.html)

Tried enigma and it worked.

[http://fuzyll.com/csaw2015/enigma](http://fuzyll.com/csaw2015/enigma)

We're given:

```
CSAW 2015 FUZYLL RECON PART 4 OF 5: Okay, okay. This isn't Engima, but the next location was "encrypted" with the JavaScript below: Pla$ja|p$wpkt$kj$}kqv$uqawp$mw>$+gwes6451+pla}[waa[ia[vkhhmj

var s = "THIS IS THE INPUT"
var c = ""
for (i = 0; i < s.length; i++) {
    c += String.fromCharCode((s[i]).charCodeAt(0) ^ 0x4);
}
console.log(c);
```

It's XOR 4 so just pass the encrypted string into it and we get:

```
The next stop on your quest is: /csaw2015/they_see_me_rollin 
```

This is the final stop:

```
CSAW 2015 FUZYLL RECON PART 5 OF 5: Congratulations! Here's your flag{I_S3ARCH3D_HI6H_4ND_L0W_4ND_4LL_I_F0UND_W4S_TH1S_L0USY_FL4G}!
```

