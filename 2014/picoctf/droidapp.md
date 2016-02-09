### Solved by superkojiman

Droid App is an 80 point forensics challenge. 

> An Android application was released for the toaster bots, but it seems like this one is some sort of debug version. Can you discover the presence of any debug information being stored, so we can plug this?
You can download the apk here.

After downloading the APK, we can decompile it using [http://www.decompileandroid.com/](http://www.decompileandroid.com). Once it's decompiled, we can download the decompiled files and unpack them. 

If we open up src/picoapp453/picoctf/com/picoapp/ToasterActivity.java we see the following:

```java
public ToasterActivity()
{  
    mystery = new String(new char[] {
        'f', 'l', 'a', 'g', ' ', 'i', 's', ':', ' ', 'w', 
        'h', 'a', 't', '_', 'd', 'o', 'e', 's', '_', 't', 
        'h', 'e', '_', 'l', 'o', 'g', 'c', 'a', 't', '_', 
        's', 'a', 'y'
    });
}
```

There's our flag: **what_does_the_logcat_say**
