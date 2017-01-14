### Solved by superkojiman 

We can store luggage and locate it later on. For instance, if we use team shoebox, and store luggage with an ID of kappakappa, we get a message:

shoebox deposited kappakappa into position 21256

It turns out that this service is vulnerable to SQL injection. If we provide the identifier we provide for the location service is:

```
a' or '1'='1
```

It actually dumps all the entries in the table: 

![](/images/2016/tjctf/luggage/01.png)

But where's the flag? The challenge description actually provides a hint:

"Fortunately, my flag went in the wrong way, so it shouldn't be difficult for you to find. Right?"

So I tried searching for "ftcjt" and found this entry:

```
(12468, u'}re1s4e_t1_3d4m_s1ht{ftcjt', u'TJCTF')
```

Reversing that string gives us the flag: tjctf{th1s_m4d3_1t_e4s1er}k
