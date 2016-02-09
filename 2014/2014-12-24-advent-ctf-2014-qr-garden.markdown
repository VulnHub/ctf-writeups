### Solved by Swappage

Why bother with programming skillz when you can get the same result with a couple of bash commands and memory greedy tools?

Qrgarden was the December 10th challenge in Advent CTF 2014: what we had to deal with was a huge image (8700 x 8700) filled with 10.000 87 x 87 qrcodes.

The objective was to determine which qrcode contained the flag, starting with "ADCTF_"

Instead of looking forward for some python libraries to deal with image cropping and qrcode reading, i decided to pursue the dumb way: imagemagick already has everything you need to crop an image, and it was a matter of a simple command

    convert qrgarden.png -crop 87x87 output/qrcode%03d.png

to get 10.000 images, each containing a different qrcode

to get the flag:

```bash
    for a in $(ls); do zbarimg $a | grep ADCTF_ >> ../flat.txt; done 
```

and then

    # cat ../flat.txt
    QR-Code:ADCTF_re4d1n9_Qrc0de_15_FuN

Sorry for this terrible writeup.
but well, it's been a busy month, and i lack time for doing CTFs and enjoying things i'd love to do.

So, yeh.. hope you'll enjoy it no matter what :)
