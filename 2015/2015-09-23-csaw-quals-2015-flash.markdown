---
layout: post
title: "CSAW Quals 2015 Flash"
date: 2015-09-23 19:17:29 -0400
author: [bitvijays]
comments: true
categories: [csaw]
---

### Solved by bitvijays

In this challenge we were provided a image(dump) file, this challenge was also pretty easy, by just running strings on the dump the flag could be found.

```
strings flash_c8429a430278283c0e571baebca3d139.img | grep flag{
flag{b3l0w_th3_r4dar}
```

The flag is **flag{b3l0w_th3_r4dar}**
