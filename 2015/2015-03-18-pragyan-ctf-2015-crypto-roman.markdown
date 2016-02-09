---
layout: post
title: "Pragyan CTF 2015 Crypto Roman"
date: 2015-03-18 08:58:55 -0400
author: [NullMode]
comments: true
categories: [pragyan]
---

### Solved by NullMode

The question2.tar.gz was downloaded an extracted and contained to files: encrypt and clue.

The file clue contained:

    #greatest roman generals of all time

...and the file encrypt contained:

    ghowdirufhdwqlw

Greatest roman generals of all time? Caesar? Was this a Caesar Cipher?

I passed the **ghowdirufhdwqlw** through [this handy tool](http://www.dcode.fr/caesar-cipher) which spat out **DELTAFORCEATNIT**. Annoyingly this was in the incorrect case. After changing the case to lower I submitted **deltaforceatnit** as the flag. Success!
