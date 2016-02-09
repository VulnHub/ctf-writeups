### Solved by bitvijays

Pickle Jar is a 30 point forensics challenge. You are provided with a file named pickle.jar.

> The police station offers free pickles to police officers. However, someone stole the pickles from the pickle jar! You find a clue on a USB drive left at the scene of the crime.

JAR (Java ARchive) is a package file format typically used to aggregate many Java class files and associated metadata and resources (text, images, etc.) into one file to distribute application software or libraries on the Java platform. It can be extracted using **_jar xf pickle.jar_** which provided two folders (COM, META-INF) and one file named pickle.p, If you read the pickle.p file it contains 
```
S'YOUSTOLETHEPICKLES'
p0 
```
The flag is **YOUSTOLETHEPICKLES**

