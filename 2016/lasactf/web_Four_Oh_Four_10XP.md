###Solved by bitvijays

We were trying to make an introductory web problem, but messed up somewhere along the way. http://web.lasactf.com:45025

The server isn't broken, I promise. But I won't promise that the page isn't hiding something. If only there was some way to look inside it!

On clicking the webpage we get
```
404 Not Found

nginx/1.8.0
```

On clicking view source we get
```

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.8.0</center>
<!-- Backup flag in case something breaks: lasactf{welc0m3_to_web_dev} -->
</body>
</html>
```

The flag is ***lasactf{welc0m3_to_web_dev}***


