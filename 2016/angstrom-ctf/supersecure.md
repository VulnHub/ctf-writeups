### Solved by z0rex

SuperSecure™ was a web challenge worth 30 points.

> Jason made a new SuperSecure™ website, but lost his password. It's displayed
> on the admin page. Can you login? 

```html
<html>
    <head>
        <title>Authentication required</title>
        <script src="//code.jquery.com/jquery-2.1.3.min.js"></script>
        <script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/rollups/sha256.js"></script>
        <script type="text/javascript">
            $(function() {
                $("#login_button").click(function() {
                    var username = $("#username").val();
                    var password = $("#password").val();
                    // Verify the inputted password, so that attackers can't get in
                    if(username == "admin" && CryptoJS.SHA256(password).toString() == "7de7b2fed84fd29656dff73bc98daef391b0480efdb0f2e3034e7598b5a412ce") {
                        // If they have the correct password go to the admin page.
                        // For extra security, the hash of the password is added into the name
                        window.location.href = "admin_" + CryptoJS.SHA256(password).toString() + ".html";
                    }
                })
            })
        </script>
    </head>
    <body>
        <div style="text-align: center;">
            <h1>Please authenticate to continue</h1>
            <br>
            <form id="login">
                <label for="username">Username</label>
                <input type="text" id="username">
                <br>
                <label for="password">Password</label>
                <input type="password" id="password">
                <br>
                <input type="button" id="login_button" value="Login!">
            </form>
        </div>
    </body>
</html>
```

The interesting part of this code is this...

```javascript
.
.
.
if(username == "admin" && CryptoJS.SHA256(password).toString() == "7de7b2fed84fd29656dff73bc98daef391b0480efdb0f2e3034e7598b5a412ce") {
    // If they have the correct password go to the admin page.
    // For extra security, the hash of the password is added into the name
    window.location.href = "admin_" + CryptoJS.SHA256(password).toString() + ".html";
}
```

... and particularily this comment

```
// For extra security, the hash of the password is added into the name
```

All we had to do was to add the has from the if statement, between `admin_` and 
`.html` so the file name looked like this

```
admin_7de7b2fed84fd29656dff73bc98daef391b0480efdb0f2e3034e7598b5a412ce.html
```

Visiting that file gave us the following message

```
Thank you for using Åweb!

The flag required for administration is all_javascript_is_open
```

Flag: `all_javascript_is_open`