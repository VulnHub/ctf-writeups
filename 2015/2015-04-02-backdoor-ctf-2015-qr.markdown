---
layout: post
title: "Backdoor CTF 2015 QR"
date: 2015-04-02 16:16:34 -0400
author: [barrebas]
comments: true
categories: [backdoor]
---

### Solved by barrebas

For this challenge, we had to decode a ton of QR images given to us via a `nc` connection for 70 points. Sounds like fun!

Upon connecting to the service, it tells us we need to decode as much flags as we can. We also get the first flag.

![](/images//2015/backdoor/qr/qr-00.png)
Right, so I turned to the solution to [QR Garden by Swappage](https://ctf-team.vulnhub.com/advent-ctf-2014-qr-garden/). I liked his approach and decided to do something similar. I installed `zbar` on my box and cobbled together a python script. That script would connect to the QR service, write out the QR code to disk using `PIL` and then call `zbarimg` to decode the QR code. I had some issues at the start because I mixed up the colors of the QR code, but then I was decoding them all pretty rapidly.

```python
from socket import *
from PIL import Image
from os import popen
import re, time, telnetlib

def readtil(delim):
    buf = b''
    while not delim in buf:
        buf += s.recv(1)
    return buf

def sendln(b):
    s.send(b + b'\n')

def getQR(qr):
    im = Image.new("RGB", (94, 94), 'white')

    pixels = im.load()
    print len(qr)
    y = 0
    x = 0
    i = 0
    while i < len(qr):
        if ord(qr[i]) == 0xe2:
            pixels[x,y] = (0, 0, 0)
            pixels[x,y+1] = (0, 0, 0)
            x += 1
        if ord(qr[i]) == 0x20:
            pixels[x,y] = (255, 255, 255)
            pixels[x,y+1] = (255, 255, 255)
            x += 1
        if ord(qr[i]) == 0xa:
            x = 0
            y += 2
        i += 1
    
    # write out QR to disk  
    im.save('qr.png')
    
    # ugly hack because ctf
    popen('zbarimg ./qr.png > qr.out')
    
    with open('qr.out') as f:
        data = f.readline()
        f.close()
    
    m = re.findall(r'Code:([0-9a-f]*)', data)
    return m[0]
    
def pwn():
    global s
    s = socket(AF_INET, SOCK_STREAM)
    s.connect(("hack.bckdr.in", 8010))

    readtil('can') # banner

    try:
        while 1:
            time.sleep(0.05)
            qr = readtil(b'\x0a                                          ')
            #print qr
            sendln(getQR(qr))
            response = s.recv(200)
            print response
    except KeyboardInterrupt:
        t = telnetlib.Telnet()
        t.sock = s
        t.interact()
        s.close()
    
pwn()
```

The script was receiving and decoding QR codes quite rapidly, but I didn't receive any flags. I finally just fired up `wireshark` to grab the flag from the TCP stream:


![](/images/2015/backdoor/qr/qr-01.png)

Not the most elegant way to grab the flag, but hey, it worked.

