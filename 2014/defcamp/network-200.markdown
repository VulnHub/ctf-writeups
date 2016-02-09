### Solved by barrebas

Again, we get a clue:

`That ****ing manager got smarter. He moved to house number 22, but we got this: ****managers.pcap`

House number 22 probably means that the ip address of this challenge is `10.13.37.21`. The pcap file contains network traffic of the manager logging in to the page at 10.13.37.21. Upon examination of this page, it seems it generates a nonce. This nonce is to encode the md5 of the password, which it then sent over as GET parameters.

Looking at the pcap file, Swappage quickly identified this:

```
user=manager&nonce=7413734ab666ce02cf27c9862c96a8e7&pass=3ecd6317a873b18e7dde351ac094ee3b
```

So we need to reverse this nonce-based encryption to get the md5'ed password of manager. The source of the login page reveals that the nonce is valid for about 100 seconds and the function that is used to generate the encoded password:

```javascript
$('.hook-submit').click(function(){
  var h1 = md5($('#pass').val());
  var h2 = $('#nonce').val();
  var xor = myxor(h1, h2);
  $('#hiddenpass').val(xor);
  setTimeout(function() { $('#form').submit(); }, 100);
});
```

This threw me off. I thought the nonce and md5 were `XOR`ed together. I couldn't be further from the truth. After wasting some time, I investigated the javascript:

```javascript
...snip...

function hex2n(c) {
	if(is_numeric(c)) return parseInt(c);
	return ord(c) - ord('a') + 10;
}

function n2hex(n) {
	if(n < 10) { return '' + n; }
	return chr(ord('a') + n - 10);
}

function myxor(h1, h2) {
	var xored = '';
	for(i = 0; i<h1.length; i++) {
		var c1 = h1.charAt(i);
		var c2 = h2.charAt(i);
		alert(hex2n(c1)); // DEBUG by barrebas
		alert(hex2n(c2)); // DEBUG by barrebas
		alert(n2hex((hex2n(c1) + hex2n(c2)) % 16)); // DEBUG by barrebas
		xored += n2hex((hex2n(c1) + hex2n(c2)) % 16);
 	}
 	return xored;
}
```
With a few carefully placed `alert()`s, I was able to figure out that the `myxor` function takes each hex character of the nonce and the md5. These values are converted to integers and then added together. After a modulus 16 the hex-representation of the result is added to the output.

I wrote a small piece of python to do the reverse operation for me, given the nonce and the encoded md5:

```python
#!/usr/bin/python

nonce = 'efa6085790a0294851202602a4833ad1'
epass = '3ecd6317a873b18e7dde351ac094ee3b'

def makeHex(c):
	return hex(c)[2:]

def getHex(c):
	return int(''.join(('0',c)), 16)

output = ''
for i in range(len(epass)):
	output += makeHex((getHex(epass[i]) - getHex(nonce[i])) % 16)

print output
```

This returned the md5 of the managers' password: `cabaf0ddf21df38cbeb77c94a40e4654`. Unfortunately, the Mighty Google did not return any hits when searching for it. Who needs it anyway? We have the md5 of the password, all we need to do is to re-encode it with a new nonce and submit it! I refreshed the login page, grabbed the nonce and ran the following piece of python:

```python
#!/usr/bin/python

nonce = 'efa6085790a0294851202602a4833ad1'
epass = '3ecd6317a873b18e7dde351ac094ee3b'
passval = 'cabaf0ddf21df38cbeb77c94a40e4654'

def makeHex(c):
	return hex(c)[2:]

def getHex(c):
	return int(''.join(('0',c)), 16)

output = ''
for i in range(len(epass)):
	output += makeHex((getHex(passval[i]) + getHex(nonce[i])) % 16)

print output
```

This returned a new, encoded md5 of the password. I copied it and entered `manager:idontcare` in the login form. I intercepted this request with TamperData, substituting the new, encoded md5 value for the old one and hit submit. Lo and behold:


```
The secret is behind bb00403ebcbfa0748bcbee426acfdb5b :)
```

Which is the md5 of `youtoo`, which thankfully was the proper flag!
