### Solved by et0x

Javascrypt is a 40 point reverse engineering challenge. 


> Tyrin Robotics Lab uses a special web site to encode their secret messages. Can you determine the value of the secret key? 

Upon visiting the link provided, you are presented with a form that encodes messages.

![](/images/2014/pico/javascrypt/01.png)

Looking at the source of the page, you see the following Javascript code:


```javascript
var key; // Global variable. 
            
// Since the key is generated when the page 
// is loaded, no one will be able to steal it
// by looking at the source! This must be secure!
function generateKey() {
  var i = 1;
  var x = 211;
  var n = 5493;
  while (i <= 25) {
    x = (x * i) % n;
    i++;
   }
  key = "flag_" + Math.abs(x);
}
            
  generateKey();
            
  // Encode the message using the 'key'
function encode() {                                                        
  var input = $("#inputmessage").val();
  var output = CryptoJS.AES.encrypt(input, key);
  $("#outputmessage").val(output);
}        
```

Now, since the solution to this problem is the key, all you have to do is find the outut to the generateKey() function.  There are plenty of ways to do this, but I just take the code and adapt it to python.

```python
#!/usr/bin/python

i = 1
x = 211
n = 5493
while (i<=25):
	x = (x*i)%n
	i += 1

print "flag_%s"%abs(x)
```

Which results in **flag_2781**
