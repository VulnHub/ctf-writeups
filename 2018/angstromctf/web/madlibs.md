### Solved by rasta_mouse and guest player RythmStick

```
When Ian was a kid, he loved to play goofy Madlibs all day long. Now, he's decided to write his own website to generate them!
```

This application is vulnerable to template injection.

![](/images/2018/angstromctf/web/madlibs-1.png)
![](/images/2018/angstromctf/web/madlibs-2.png)

Part of the source is available to us, where we see:

```
app = Flask(__name__)
app.secret_key = open("flag.txt").read()
```

and also that there is a 12 character limit in the author field.

```
authorName = inpValues.pop(0)[:12]
```

To get the flag, enter `{{config}}` in the `author` field.

`actf{wow_ur_a_jinja_ninja}`