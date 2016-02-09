---
layout: post
title: "PoliCTF 2015 John The Dropper"
date: 2015-07-12 17:19:12 -0400
author: [barrebas,superkojiman, swappage]
comments: true
categories: [polictf]
---

### Solved by barrebas, superkojiman, and Swappage

John the Dropper was an interesting 100 point challenge. I'd love to see how it was implemented!

<!--more-->

We're given a host, `dropper.polictf.it`. It has no open ports, but the challenge description mentioned that John did not need ports to communicate. I left this challenge for a while, focusing on others. When I got back, superkojiman noticed that pinging this host dropped a lot of packets. He saw patterns: sometimes one packet dropped, sometimes three in a row. This made me think of Morse immediately. 

I started pinging the host and grabbed the output of `ping`:

```
PING dropper.polictf.it (52.18.119.20) 56(84) bytes of data.
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=1 ttl=50 time=23.8 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=3 ttl=50 time=24.2 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=5 ttl=50 time=24.1 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=7 ttl=50 time=23.8 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=8 ttl=50 time=23.9 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=12 ttl=50 time=23.8 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=16 ttl=50 time=24.2 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=20 ttl=50 time=23.9 ms
64 bytes from ec2-52-18-119-20.eu-west-1.compute.amazonaws.com (52.18.119.20): icmp_req=21 ttl=50 time=24.0 ms
...snip...
```

As you can see, request 2, 4 and 6 are dropped. With the following one-liner, I extracted the icmp_req numbers:

```bash
cat pings.txt |awk '{print $6}' |awk '{split($0,a,"="); print a[2]}' > sequence.txt
```

I wrote a python script to translate the drops to Morse:

```python
seq = []

with open('sequence.txt') as f:
    lines = f.readlines()
    for l in lines:
        seq.append(int(l))
    f.close()

msg = ""
for i in range(len(seq)-2):
    if seq[i+1] - seq[i] - 1 == 0:
        count += ' '
    if seq[i+1] - seq[i] - 1 == 1:
        msg += '.'
    if seq[i+1] - seq[i] - 1 == 3:
        msg += '-'
print msg
```

Which yielded:

```
... --- ...    - .... .. ...    .. ...    - .... .    ..-. .-.. .- --. .--.- .. - -....- .. ... -....- -. . ...- . .-. -....- - --- --- -....- .-.. .- - . -....- ..-. --- .-. -....- .- -....- -.. .-. --- .--. .--.--   
```

The first three characters spell out "SOS". I translated the rest by hand and found: `SOS THIS IS THE FLAG?IT?IS?NEVER?TOO?LATE?FOR?A?DROP?`. I couldn't really figure out the characters that are marked `?`. I guessed them to be underscores, but in the end, [duckduckgo](https://duckduckgo.com/?q=...+---+...++++-+....+..+...++++..+...++++-+....+.++++..-.+.-..+.-+--.+.--.-+..+-+-....-+..+...+-....-+-.+.+...-+.+.-.+-....-+-+---+---+-....-+.-..+.-+-+.+-....-+..-.+---+.-.+-....-+.-+-....-+-..+.-.+---+.--.+.--.--++++morse&ia=answer) came to the rescue. The final flag was lowercase: `flag{it-is-never-too-late-for-a-drop}`. 

