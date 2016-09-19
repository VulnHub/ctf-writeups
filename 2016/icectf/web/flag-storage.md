### Solved by superkojiman

We're supposed to get the password from [http://flagstorage.vuln.icec.tf/](http://flagstorage.vuln.icec.tf/), which provides a form to enter in a username and password. The challenge hints that this is a SQL injection vulnerability. Sure enough, entering the following for the username returns the flag:

```
admin' or '1'='1' --
```

Flag: IceCTF{why_would_you_even_do_anything_client_side}
