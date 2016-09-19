### Solved by superkojiman

The description hints that we need to look at the domain's text entry. 

```
$ nslookup -type=txt search.icec.tf
Server:     209.222.18.222
Address:    209.222.18.222#53

Non-authoritative answer:
search.icec.tf  text = "IceCTF{flag5_all_0v3r_the_Plac3}"

Authoritative answers can be found from:
```

Done. 
