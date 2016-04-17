### Solved by z0rex

"Java is the Best" was a reversing challenge worth 50 points.

> What kind of input makes this program happy? 

I decompiled the `.class` file with `jad`

```
# jad SuperSecure.class 
Parsing SuperSecure.class...The class file version is 51.0 (only 45.3, 46.0 and 47.0 are supported)
 Generating SuperSecure.jad
```

Looking at the decompiled I would read the flag directly.

```
# cat SuperSecure.jad 
// Decompiled by Jad v1.5.8e. Copyright 2001 Pavel Kouznetsov.
// Jad home page: http://www.geocities.com/kpdus/jad.html
// Decompiler options: packimports(3) 
// Source File Name:   SuperSecure.java

import java.io.PrintStream;

public class SuperSecure
{

    public SuperSecure()
    {
    }

    public static void main(String args[])
    {
        if(args[0].charAt(0) == 'd' && args[0].charAt(1) == 'o' && args[0].charAt(2) == 'n' && args[0].charAt(3) == 't' && args[0].charAt(4) == '_' && args[0].charAt(5) == 'u' && args[0].charAt(6) == 's' && args[0].charAt(7) == 'e' && args[0].charAt(8) == '_' && args[0].charAt(9) == 'j' && args[0].charAt(10) == 'a' && args[0].charAt(11) == 'v' && args[0].charAt(12) == 'a' && args[0].charAt(13) == '_' && args[0].charAt(14) == 'i' && args[0].charAt(15) == 'f' && args[0].charAt(16) == '_' && args[0].charAt(17) == 'y' && args[0].charAt(18) == 'o' && args[0].charAt(19) == 'u' && args[0].charAt(20) == '_' && args[0].charAt(21) == 'w' && args[0].charAt(22) == 'a' && args[0].charAt(23) == 'n' && args[0].charAt(24) == 'n' && args[0].charAt(25) == 'a' && args[0].charAt(26) == '_' && args[0].charAt(27) == 'h' && args[0].charAt(28) == 'i' && args[0].charAt(29) == 'd' && args[0].charAt(30) == 'e' && args[0].charAt(31) == '_' && args[0].charAt(32) == 'c' && args[0].charAt(33) == 'o' && args[0].charAt(34) == 'd' && args[0].charAt(35) == 'e')
            System.out.println("Hooray!");
    }
}
```

Flag: dont_use_java_if_you_wanna_hide_code