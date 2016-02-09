---
layout: post
title: "Boston Key Party 2015 Symphony"
date: 2015-03-18 09:14:51 -0400
author: [barrebas]
comments: true
categories: [boston key party]
---

### Solved by barrebas

We knew BKP would be tough, but they are asking us to find a SHA1 hash collision!

```php
if (isset($_GET['name']) and isset($_GET['password'])) {
    if ($_GET['name'] == $_GET['password'])
         print 'Your password can not be your name.';
    else if (sha1($_GET['name']) === sha1($_GET['password']))
        die('Flag: '.$flag);
    else
        print '<p class="alert">Invalid password.</p>';
}
```

SHA1 is *theoretically* broken, but practically not so much. Instead, we again rely on passing parameters as arrays. We set `name` to `[a, b]` and `password` to `[a]`. This will bypass the first equality check. The `sha1` function will then take the first entry of the array and generate the hashes of `a`, which, unsurprisingly, are equal!

![](/images/2015/bkp/symphony/symphony-flag.png)


