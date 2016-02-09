---
layout: post
title: "PoliCTF 2015 John The Referee"
date: 2015-07-12 17:08:52 -0400
author: [swappage, nullmode]
comments: true
categories: [polictf]
---

### Solved by Swappage and NullMode

John the referee was a 150 points web challenge with some crypto added to the recipe :)

The web application looked like a shop selling different types of tshirts and our objective was to discover an hidden item in the shop.

<!-- more -->

{% img center /images/2015/polictf/john_the_referee/index.png %}

The search function looked interesting, because once a search string was submitted, the user was redirected to a search result page having an URL similar to the following:

    http://referee.polictf.it/search-result/e25fc408c9ac78c3fb02ff08ece71f1b4182747916789ca739281400a7a33cca

Obviously trying to perform a basic SQL injection attack revealed that the input from the search form was sanitized.

{% img center /images/2015/polictf/john_the_referee/sanitized.png %}

But let's get back to the search result page.
From a first look the last part of the URL looked like an hash (a sha256), but messing around with the search form, and submitting longer search keywords, we could notice that the hexdecimal string was expanding by 32 characters (16 bytes) every 16 characters in the search string.

This suggested that instead of an hash, it was most likely a block cipher encoded text, and chances that it was containing our search string were quite high.

Also by searching the same string multiple time, the encrypted string in the URL was different every time, and this meant that an initialization vector was used.

Putting all the things togeder, from what we knew the URL hex encoded string was most likely an AES256 CBC encrypted version of our search keyword.

This meant that if we could manipulate the content of the encrypted text directly from the URL, we could probably bypass the sanitization and perform a SQL injection on the backend database.

There are two types of attacks that are especially effective against block ciphers encrypted data:

- padding oracle
- bit flipping.

I spent quite a while attempting a badding oracle attack, but without any success, so i decided to try a bit flipping one.

{% img center /images/2015/polictf/john_the_referee/goodresult.png %}

A good search result page looked like the one from the above picture, this suggested that probably a union select injection would be effective

The only sanitized characters were \ ' " and a few others, so i decided to submit the following search string from the search page

    & union select 1,2,3,4 ;#--

and start messing with the search result ciphertext

{% img center /images/2015/polictf/john_the_referee/union_inject.png %}

Soon enaugh we had a working injection :)

From there on, i continued with some standard enumeration techniques to determine the database and the table name; fairly long and tedious part, i really wanted to prevent the usage of banned characters in the query, and therefore i couldn't use any *where* clause.

Still at the end i figured out that the database name was johnthereferee and the table name was uniforms.

at that point running the following query returned the flag.

{% img center /images/2015/polictf/john_the_referee/flag.png %}

