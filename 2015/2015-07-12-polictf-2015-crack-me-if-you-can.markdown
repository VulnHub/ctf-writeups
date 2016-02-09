### Solved by barrebas

Crack Me If You Can involved one of my least favorite things: Android APKs!

<!--more-->

I downloaded the APK and directly uploaded it to [decompileandroid.com](http://www.decompileandroid.com/). Among the decompiled files I found `src/it/politctf/LoginActivity.java` and three other java files. After inspecting `LoginActivity.java`, I found this function:

```java
private boolean a(String s)
    {
        if (s.equals(c.a(it.polictf2015.b.a(it.polictf2015.b.b(it.polictf2015.b.c(it.polictf2015.b.d(it.polictf2015.b.g(it.polictf2015.b.h(it.polictf2015.b.e(it.polictf2015.b.f(it.polictf2015.b.i(c.c(c.b(c.d(getString(0x7f0c0038))))))))))))))))
        {
            Toast.makeText(getApplicationContext(), getString(0x7f0c003c), 1).show();
            return true;
        } else
        {
            return false;
        }
    }
```

Interesting. It performs a bunch of operations on a string, which I don't know yet. However, on of the operations is this one:

```java
    public static String c(String s)
    {
        return s.replace("buga", "Goo");
    }
```

So I did the following:


```bash
$ strings crack-me-if-you-can.apk  |grep buga
ee[[c%l][c{g}[%{\%Mc%spdgj=]T%aat%=O%bRu%sc]c%ti[o%n=Wcs%=No[t=T][hct%=buga[d=As%=W]e=T%ho[u%[%g]h%t[%}%
```

I now had the string and all the operations on the string:

```java
public class b
{

    public static String a(String s)
    {
        return s.replace("c", "a");
    }

    public static String b(String s)
    {
        return s.replace("%", "");
    }

    public static String c(String s)
    {
        return s.replace("[", "");
    }

    public static String d(String s)
    {
        return s.replace("]", "");
    }

    public static String e(String s)
    {
        return s.replaceFirst("\\{", "");
    }

    public static String f(String s)
    {
        return s.replaceFirst("\\}", "");
    }

    public static String g(String s)
    {
        return s.replaceFirst("c", "f");
    }

    public static String h(String s)
    {
        return s.replaceFirst("R", "f");
    }

    public static String i(String s)
    {
        return s.replace("=", "_");
    }
}

public class c
{

    public static String a(String s)
    {
        return s.replace("aa", "ca");
    }

    public static String b(String s)
    {
        return s.replace("aat", "his");
    }

    public static String c(String s)
    {
        return s.replace("buga", "Goo");
    }

    public static String d(String s)
    {
        return s.replace("spdgj", "yb%e");
    }
}
```

Using this string, I started working my way back, applying all the operations of `a.java`, `b.java` and `c.java` by hand. Finally, I ended up with the string `flag{Maybe_This_Obfuscation_Was_Not_That_Good_As_We_Thought}`. 

