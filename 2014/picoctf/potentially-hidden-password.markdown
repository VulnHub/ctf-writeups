### Solved by superkojiman

Potentially Hidden Password is a 100 point web exploitation challenge. 

> This Daedalus Corp. website loads images in a rather odd way... [Source Code]

Here's the webpage:

![](/images/2014/pico/potentially_hidden_password/01.png)

Looking at the source reveals the location of the flag: 

```php
  <?php
     $config_file = fopen("/resources/config/admin_mode.config", "r");
     if (fgets($config_file) === "true") {
        $flag_file = fopen("/resources/secrets/flag", "r");
        echo fgets($flag_file);
        flose($flag_file);
     }
     fclose($config_file);
       ?>
</div>
```

The description says the images are loaded in a strange way. Let's have a look: 

```
<center>
  <img src="file_loader.php?file=zone1.jpg" class="zone"/>
  <img src="file_loader.php?file=zone2.jpg" class="zone"/>
  <img src="file_loader.php?file=zone3.jpg" class="zone"/><br><br>
  <img src="file_loader.php?file=zone4.jpg" class="zone"/>
  <img src="file_loader.php?file=zone5.jpg" class="zone"/>
  <img src="file_loader.php?file=zone6.jpg" class="zone"/><br>
</center>
```

Smells like local file inclusion. Let's capture the request in BurpSuite's proxy mode and change one of the image files to flag:

![](/images/2014/pico/potentially_hidden_password/02.png)

We get the following response: 

![](/images/2014/pico/potentially_hidden_password/03.png)

Now we know what the current directory is. Let's do this again but specify ../secrets/flag instead of just flag

![](/images/2014/pico/potentially_hidden_password/04.png)

The flag is **i_like_being_included**
