### Solved by rasta_mouse

```
Your goal hasn't changed. Log in.
```

Similar to `Source Me 1`, we have to login.

![](/images/2018/angstromctf/web/source-me-2.png)

We can see this javascript in the source:

```
<script src="./md5.js"></script>
<script>
	var checkLogin = function () {
		var password = document.getElementById("password").value;
		if (document.getElementById("uname").value != "admin"){
			console.log(uname);
			document.getElementById("message").innerHTML = "Only admin user allowed!";
			return;
		} else {
			var passHash = md5(password);
			if (passHash == "bdc87b9c894da5168059e00ebffb9077"){
				window.location = "./login.php?user=admin&pass=" + password;
			} else {
				document.getElementById("message").innerHTML = "Incorrect Login";
			}
		}
		return;
	}
</script>
```

`bdc87b9c894da5168059e00ebffb9077` is the md5 hash of `password1234`.

Login with `admin:password1234` for the flag.

`actf{md5_hash_browns_and_pasta_sauce}`