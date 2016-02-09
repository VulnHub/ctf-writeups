---
layout: post
title: "Boston Key Party 2015 Museum"
date: 2015-03-18 09:05:35 -0400
author: [barrebas]
comments: true
categories: [boston key party]
---

### Solved by barrebas

This challenge required us to input the next number in a seemingly random sequence of numbers. Upon inspection of the source code, the random numbers are not so random at all: the seed value is limited to roughly 20,000. I wrote a small script in `php` to generate the list of numbers for each seed value:

```php
<?php

for ($i=0; $i<20000; $i++){
    mt_srand($i);
    $out = "";
    for ($n=0; $n<4; $n++) {
        $out = $out . mt_rand(0, 0xffffff) . " ";
    }
    print $out . "\n";
}
?>
```

This generated an output file, which we could `grep`:

![](/images/2015/bkp/museum/museum-rng-broken.png)

and this gave us the flag:

![](/images/2015/bkp/museum/museum-flag.png)

