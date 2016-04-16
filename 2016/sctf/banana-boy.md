### Solved by z0rex

"Banana boy" was a forensics challenge worth 20 points.

> Carter loves bananas, but I heard a rumor that he's hiding something!  
> Can you find it?

I downloaded the attached file `carter.jpg` and took a quick look at it.

![Image](/images/2016/sctf/banana-boy/carter.jpg)

Next stage was to have a look at it with `binwalk`

```
# binwalk carter.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
382           0x17E           Copyright string: "Copyright (c) 1998 Hewlett-Packard Company"
3192          0xC78           TIFF image data, big-endian, offset of first image directory: 8
140147        0x22373         JPEG image data, JFIF standard 1.01
140177        0x22391         TIFF image data, big-endian, offset of first image directory: 8
```

It's easy to see here that there's another image stored at offset `140147`. So
all that had to be done was to extract this image. Since I've hardly done any
forensics challenges before, I turned to my team and asked for helping hand,
which I got right away. Thanks Swappage!

```
# dd if=carter.jpg of=carter-1.jpg skip=140147 bs=1
54041+0 records in
54041+0 records out
54041 bytes (54 kB, 53 KiB) copied, 0.118547 s, 456 kB/s
```

This gave me the following image.

![Image](/images/2016/sctf/banana-boy/flag.jpg)

Flag: `sctf{tfw_d4nk_m3m3s_w1ll_a1w4y5_pr3v4il}`