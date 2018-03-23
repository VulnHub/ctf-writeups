### Solved by rasta_mouse

```
Get me! Over here.
```

![](/images/2018/angstromctf/web/get-me-1.png)

Clicking on the button sends a GET request to `http://web.angstromctf.com:3005/?auth=false`.
The output is `Hey, you're not authorized!`

Changing the `auth` parameter to `true` gets us the flag.

`actf{why_did_you_get_me}`