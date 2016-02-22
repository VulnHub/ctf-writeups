### Solved by superkojiman

We're given a URL that points to a simple static HTML blog. There's a hint about git in the blog, and as it turns out, there's a git repository at https://0ldsk00lblog.ctf.internetwache.org/

We can't view directory index listing, but we can view individual known files in each directory. The trick to solving this seemed to be in grabbing the .git repository. Although wget wouldn't work for that, [DVCS-Pillage](https://github.com/evilpacket/DVCS-Pillage) would. Once I had grabbed the repository, I took a peek at the commit logs:

```
# git log
commit 8c46583a968da7955c13559693b3b8c5e5d5f510
Author: Sebastian Gehaxelt <github@gehaxelt.in>
Date:   Fri Jan 22 02:58:01 2016 +0100

    My recent blogpost

commit 26858023dc18a164af9b9f847cbfb23919489ab2
Author: Sebastian Gehaxelt <github@gehaxelt.in>
Date:   Fri Jan 22 02:57:44 2016 +0100

    Added another post

commit dba52097aba3af2b30ccbc589912ae67dcf5d77b
Author: Sebastian Gehaxelt <github@gehaxelt.in>
Date:   Fri Jan 22 02:55:33 2016 +0100

    Added next post

commit 14d58c53d0e70c92a3a0a5d22c6a1c06c4a2d296
Author: Sebastian Gehaxelt <github@gehaxelt.in>
Date:   Fri Jan 22 02:55:11 2016 +0100

    Initial commit
```

I did a diff on each one, and found that commit 26858023dc18a164af9b9f847cbfb23919489ab2 contained the flag:

```
# git diff 26858023dc18a164af9b9f847cbfb23919489ab2
diff --git a/index.html b/index.html
index 5508adb..25a3f35 100644
--- a/index.html
+++ b/index.html
@@ -3,9 +3,15 @@
        <title>0ldsk00l</title>
 </head>
 <body>
-       <h2>2000</h2>
+
+       <h1>Welcome to my 0ldsk00l blog.</h1>
+       <p>
+               Now this is some oldskool classic shit. Writing your blog manually without all this crappy bling-bling CSS / JS stuff.
+       </p>
+
+       <h2>2016</h2>
        <p>
-               Oh, did I say that I like kittens? I like flags, too: IW{G1T_1S_4W3SOME}
+               It's 2016 now and I need to somehow keep track of my changes to this document as it grows and grows. All people are talking about a tool called 'Git'. I think I might give this a try.
        </p>
 
        <h2>1990-2015</h2>
```

The flag: IW{G1T_1S_4W3SOME}
