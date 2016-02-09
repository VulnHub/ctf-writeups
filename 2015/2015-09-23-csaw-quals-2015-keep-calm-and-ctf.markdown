---
layout: post
title: "CSAW Quals 2015 Keep Calm and CTF"
date: 2015-09-23 19:17:29 -0400
author: [bitvijays]
comments: true
categories: [csaw]
---

### Solved by bitvijays

In this forensic challenge, we were provided with and image which said "Keep Calm and CTF". This challenge was pretty easy, we could solve it just by using file command or exiftool on the image.

```
file img.jpg 
img.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, Exif Standard: [TIFF image data, big-endian, direntries=3, resolutionunit=2, copyright=h1d1ng_in_4lm0st_pla1n_sigh7], baseline, precision 8, 600x700, frames 3
```

The flag is **h1d1ng_in_4lm0st_pla1n_sigh7**
