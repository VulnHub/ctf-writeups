---
layout: post
title: "Eko Party Pre-CTF 2015 Reversing Psychology"
date: 2015-09-20 23:36:31 -0400
author: [superkojiman]
comments: true
categories: [ekoparty]
---

### Solved by superkojiman

This is an odd one for a 200 point challenge. We have a 32-bit binary but running it causes a SIGILL. I loaded it up in IDA and it complained that there were too many nodes to display and that I could change it in Options. So I changed it to display 10,000 nodes and the flag appeared in the graph overview: 

![](/images/2015/ekoparty/revpsycho/flag.png)

Flag is **EKO{R3v3rs1ng_Psych0}**
