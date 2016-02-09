###Solved by bitvijays

This post provides the write up of Android Section available in Pragyan CTF.

<strong>Challenge 1: Hackerz</strong>

>Adam and eve meet in garden, snake gave eve an apple, that was the first SYN.

In this android challenge, we were provided a apk file with the information that a SYN might refer to a network activity. Recently, a paper <a href="http://dl.packetstormsecurity.net/papers/attack/intro-android-malware.pdf">Introduction to Malware Analysis</a> was published on Packetstorm. Following that, we check the apk file using <a href="https://anubis.iseclab.org/">Anubis</a>. Analysed the circle.apk file analysis available @ <a href="https://anubis.iseclab.org/?action=result&task_id=111cd3c141e1ecaf4dd3d3000573f9611&format=html">Circle APK</a>.

In the network activity, we see a base64 encoded string "eTB1XzRyM180X2g0Y2szcg==", which 

```
echo "eTB1XzRyM180X2g0Y2szcg==" | base64 -d
y0u_4r3_4_h4ck3r
```

The flag is **y0u_4r3_4_h4ck3r**


<strong>Challenge 2: Fast and Furious</strong>

In the second android challenge, we were provided a fast.apk file which when analyzed on Anubis, didn't provide any useful information. Time to extract the java files. We can use <a href="http://www.decompileandroid.com/">Decompile Android</a> to get the Java and smali source code.

In main.java file, interesting string is present
```
 String s = a("65544231587a52794d3138316458417a636c396d4e44553343673d3d");
```

Let's convert this hex string to ascii
```
"65544231587a52794d3138316458417a636c396d4e44553343673d3d".decode("hex")
'eTB1XzRyM181dXAzcl9mNDU3Cg=='

echo "eTB1XzRyM181dXAzcl9mNDU3Cg==" | base64 -d
y0u_4r3_5up3r_f457
```

The flag is **y0u_4r3_5up3r_f457**

