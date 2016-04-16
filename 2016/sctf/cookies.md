### Solved by z0rex

Cookies was a reversing challenge worth 75 points.

> My mom put a password on the cookie jar :(  
> Will you help me get a cookie?

I downloaded the `jar` file and decompiled it with http://www.javadecompilers.com/

Looking at the source code I found something interesting

```java
MessageDigest digest = MessageDigest.getInstance("MD5");
byte[] arr = digest.digest(new String(this.passwordField1.getPassword()).getBytes("UTF-8"));
String md5 = new BigInteger(1, arr).toString(16);
System.out.println(md5);
String answer = "fdf87a05e2169b88a8db5a1ebc15fa50";
if (md5.equals(answer)) {
    System.out.println("success! it's working!");
}
```

`String answer = "fdf87a05e2169b88a8db5a1ebc15fa50";`

Checking that hash against hashkiller.co.uk I found this `thisisaverystrongpassword`.

With this information I wanted to see if it was enough to get the flag.

![Screenshot 01.png](/hackpwn/writeups/wiki/images/CTF/2016/sCTF/cookies/01.png)

![Screenshot 02.png](/hackpwn/writeups/wiki/images/CTF/2016/sCTF/cookies/02.png)


Flag: `sctf{g3t_y0ur_h4nd_0ut_0f_my_c00k13_j4r!}`