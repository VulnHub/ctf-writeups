### Solved by bitvijays
This Misc challenge was very easy, we were provided with a number in hex which needs to be converted in to ascii.

>Today is warmup.

0x41444354465f57334c43304d335f37305f414443374632303134

This can be achieved very easily by python.

``` Python
>>> "41444354465f57334c43304d335f37305f414443374632303134".decode('hex')
'ADCTF_W3LC0M3_70_ADC7F2014'

Or

>>> import binascii     
>>> binascii.unhexlify("41444354465f57334c43304d335f37305f414443374632303134")
'ADCTF_W3LC0M3_70_ADC7F2014'
```

***The flag is ADCTF_W3LC0M3_70_ADC7F2014***
