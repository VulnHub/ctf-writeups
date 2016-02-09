---
layout: post
title: "Backdoor CTF 2015 TIM"
date: 2015-04-02 16:11:17 -0400
author: [superkojiman]
comments: true
categories: [backdoor]
---

### Solved by superkojiman

Clone the repository at https://github.com/backdoor-ctf/TIM/ and view the git log. We see two characters for each commit. Join these and we get:

```
47414c4620316168732074696d6d6f63206863616520666f2073726574636172616863206f7774207473726966206e696f4a 
```
 
Convert to ascii and we get the backward string: GALF 1ahs timmoc hcae fo sretcarahc owt tsrif nioJ 
 
Which reads: Join first two characters of each commit sha1 FLAG 

We take the first two byte of each SHA1 commit and concatenate them. Doing so returns the flag padded with zeroes:

```
000000000000000000flag_goes_here_flag_goes_here_flag_goes_here_flag_goes_here_flag000000000000000000d4
```
