---
layout: post
title: "Eko Party Pre-CTF BASE Unknown"
date: 2015-09-20 23:26:02 -0400
author: [superkojiman]
comments: true
categories: [ekoparty]
---

### Solved by superkojiman

We're given the following string:

```
IVFU662CIFJUKXZTGJPWG2DBNRWH2===
```

Looks like it's Base64 encoded at first, but it's not. It's actually Base32 encoded. Decode it to get EKO{BASE_32_chall}

