### Solved by barrebas

This web challenge sports another login that we need to bypass. The source code is given and features this password check:

```php
if (isset($_GET['password'])) {
    if (strcmp($_GET['password'], $flag) == 0)
         die('Flag: '.$flag);
    else
         print '<p class="alert">Invalid password.</p>';
}
```

So we'll need to bypass the `strcmp`. Luckily, stackoverflow has the answer:

![](/images/2015/bkp/northeastern_university/university-stackoverflow.png)

The trick here is to make password an array. This will result in `strcmp` returning NULL, thus passing the comparison and giving us the flag:

![](/images/2015/bkp/northeastern_university/university-flag.png)

