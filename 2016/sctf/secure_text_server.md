### Solved by superkojiman

This is a 60 point reversing challenge. We're given a jar file called Secure_Text_Saver.jar. I uploaded it to [http://www.javadecompilers.com/](http://www.javadecompilers.com/) and found that LoginPage.java contained hardcoded credentials in the main() function:

```java
    public static void main(String[] args) {
        accounts.add(new Account("ztaylor54", "]!xME}behA8qjM~T".toCharArray()));
```

Logging in with these credentials returned the flag.

![](/images/2016/sctf/secure_text_server/01.png)
