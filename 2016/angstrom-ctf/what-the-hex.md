### Solved by z0rex

"What the Hex" was a crypto challenge worth 15 points.

> Decode using hex and see what you get...  
> 6236343a20615735305a584a755a58526659323975646d567963326c76626c3930623239736331397962324e72 

Only needed my terminal for this.

First I decoded the hex encoded string.

```
$ echo "6236343a20615735305a584a755a58526659323975646d567963326c76626c3930623239736331397962324e72" | xxd -r -p
b64: aW50ZXJuZXRfY29udmVyc2lvbl90b29sc19yb2Nr
```

Then I decoded the base64 encoded string

```
$ echo aW50ZXJuZXRfY29udmVyc2lvbl90b29sc19yb2Nr | base64 -d
internet_conversion_tools_rock
```

Flag: `internet_conversion_tools_rock`