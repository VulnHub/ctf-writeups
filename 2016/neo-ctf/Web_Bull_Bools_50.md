###Solved by bitvijays 

Bull Bools
50

How bout that picture of that bull??

162.243.0.171/Bull.php

On clicking view source:
```

<html>
<head> <title>Bull Bools</title>
<link rel="icon" href="https://neoctf.ctfd.io/static/img/favicon.ico" type="image/x-icon">
<style>
#Get {
display: none;
}
</style>
</head>
<body>
<h3> Nothing suspicious about your bull ;) </h3>
<img src=''></img>
<form action="Bull.php" method="GET">
	<input id='Get' name="Get_Flag" value="false">
	<input class="btn btn-primary" type="submit" value="Clicking Here may help you :)">
</form>
</body>
```

On clicking the button, the http request is "http://162.243.0.171/Bull.php?Get_Flag=false"

changing Get_Flag=true give the flag

The flag is **flag{G0t_D@T_F1@g**
