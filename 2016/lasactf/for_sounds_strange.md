### Solved by superkojiman

This is a forensics challenge. We're given an audio file and a Java source file that "encrypts" a message into the audio file. The encryption method is provided, but the decryption is redacted. Using the available information, I put together a test Java program that would encrypt my own plaintext so I could examine the output file:

```java
import java.io.*;
public class Test {
    private static void encryptAudio(String toEncrypt, FileOutputStream out) throws IOException
    {
        byte[] bytes = toEncrypt.getBytes();
        AudioInputStream data = new AudioInputStream(new ByteArrayInputStream(bytes), new AudioFormat(2000f, 8, 1, false, false), bytes.length);
        AudioSystem.write(data, AudioFileFormat.Type.AIFF, out);
    }
    public static void main(String[] args) {
        try {
            FileOutputStream out = new FileOutputStream(new File("my_sample.aiff"));
            encryptAudio("AAAABBBBCCCCDDDD", out);
        } catch (Exception e) {
        }
    }
}
```

Compiling and running the program generates a sound file called my_sample.aiff. Running this under `xxd` revealed the following:

```
# javac Test.java && java Test
# xxd -g1 my_sample.aiff
00000000: 46 4f 52 4d 00 00 00 3e 41 49 46 46 43 4f 4d 4d  FORM...>AIFFCOMM
00000010: 00 00 00 12 00 01 00 00 00 10 00 08 40 09 fa 00  ............@...
00000020: 00 00 00 00 00 00 53 53 4e 44 00 00 00 18 00 00  ......SSND......
00000030: 00 00 00 00 00 00 c1 c1 c1 c1 c2 c2 c2 c2 c3 c3  ................
00000040: c3 c3 c4 c4 c4 c4                                ......
```

The "c1 c1 c1 c1 c2 c2 c2 c2 ..." was clearly my encrypted string. The difference between the encrypted byte and the plaintext was 0x80, so it made sense that to decrypt it, I just had to subtract 0x80 from the encrypted byte. 

Here's what the sounds_strange.aiff looks like:

```
# xxd -g1 sounds_strange.aiff
00000000: 46 4f 52 4d 00 00 01 13 41 49 46 46 43 4f 4d 4d  FORM....AIFFCOMM
00000010: 00 00 00 12 00 01 00 00 00 e5 00 08 40 09 fa 00  ............@...
00000020: 00 00 00 00 00 00 53 53 4e 44 00 00 00 ed 00 00  ......SSND......
00000030: 00 00 00 00 00 00 c9 a0 f3 f0 e5 ee f4 a0 f3 ef  ................
00000040: a0 ed f5 e3 e8 a0 f4 e9 ed e5 a0 ed e1 eb e9 ee  ................
00000050: e7 a0 f4 e8 e9 f3 a0 f3 ef ee e7 a0 e6 ef f2 a0  ................
00000060: f9 ef f5 a1 a0 c9 a0 e8 ef f0 e5 a0 f9 ef f5 a0  ................
00000070: ec e9 eb e5 a0 e9 f4 a1 a0 d7 e8 e1 f4 bf a0 d9  ................
00000080: ef f5 a0 e4 ef ee a7 f4 bf a0 c9 e6 a0 f9 ef f5  ................
00000090: a0 e4 ef ee a7 f4 a0 e1 f0 f0 f2 e5 e3 e9 e1 f4  ................
000000a0: e5 a0 f4 e8 e5 a0 f1 f5 e1 ec e9 f4 f9 a0 ef e6  ................
000000b0: a0 ed f9 a0 ed f5 f3 e9 e3 a0 f4 e8 e5 ee a0 f9  ................
000000c0: ef f5 a0 e3 e1 ee a0 ea f5 f3 f4 a0 ec e5 e1 f6  ................
000000d0: e5 ae a0 c1 ee e4 a0 e4 ef ee a7 f4 a0 e6 ef f2  ................
000000e0: e7 e5 f4 a0 f4 ef a0 f4 e1 eb e5 a0 f9 ef f5 f2  ................
000000f0: a0 e6 ec e1 e7 a0 f7 e9 f4 e8 a0 f9 ef f5 ba a0  ................
00000100: ec e1 f3 e1 e3 f4 e6 fb f4 e8 b3 df f3 b0 f5 ee  ................
00000110: e4 f3 df b0 e6 df f4 b3 f8 f4 fd                 ...........
```

I wrote the following python script to decrypt the message:

```python
#!/usr/bin/env python
s = [
0xc9, 0xa0, 0xf3, 0xf0, 0xe5, 0xee, 0xf4, 0xa0, 0xf3, 0xef,
0xa0, 0xed, 0xf5, 0xe3, 0xe8, 0xa0, 0xf4, 0xe9, 0xed, 0xe5, 0xa0, 0xed, 0xe1, 0xeb, 0xe9, 0xee,
0xe7, 0xa0, 0xf4, 0xe8, 0xe9, 0xf3, 0xa0, 0xf3, 0xef, 0xee, 0xe7, 0xa0, 0xe6, 0xef, 0xf2, 0xa0,
0xf9, 0xef, 0xf5, 0xa1, 0xa0, 0xc9, 0xa0, 0xe8, 0xef, 0xf0, 0xe5, 0xa0, 0xf9, 0xef, 0xf5, 0xa0,
0xec, 0xe9, 0xeb, 0xe5, 0xa0, 0xe9, 0xf4, 0xa1, 0xa0, 0xd7, 0xe8, 0xe1, 0xf4, 0xbf, 0xa0, 0xd9,
0xef, 0xf5, 0xa0, 0xe4, 0xef, 0xee, 0xa7, 0xf4, 0xbf, 0xa0, 0xc9, 0xe6, 0xa0, 0xf9, 0xef, 0xf5,
0xa0, 0xe4, 0xef, 0xee, 0xa7, 0xf4, 0xa0, 0xe1, 0xf0, 0xf0, 0xf2, 0xe5, 0xe3, 0xe9, 0xe1, 0xf4,
0xe5, 0xa0, 0xf4, 0xe8, 0xe5, 0xa0, 0xf1, 0xf5, 0xe1, 0xec, 0xe9, 0xf4, 0xf9, 0xa0, 0xef, 0xe6,
0xa0, 0xed, 0xf9, 0xa0, 0xed, 0xf5, 0xf3, 0xe9, 0xe3, 0xa0, 0xf4, 0xe8, 0xe5, 0xee, 0xa0, 0xf9,
0xef, 0xf5, 0xa0, 0xe3, 0xe1, 0xee, 0xa0, 0xea, 0xf5, 0xf3, 0xf4, 0xa0, 0xec, 0xe5, 0xe1, 0xf6,
0xe5, 0xae, 0xa0, 0xc1, 0xee, 0xe4, 0xa0, 0xe4, 0xef, 0xee, 0xa7, 0xf4, 0xa0, 0xe6, 0xef, 0xf2,
0xe7, 0xe5, 0xf4, 0xa0, 0xf4, 0xef, 0xa0, 0xf4, 0xe1, 0xeb, 0xe5, 0xa0, 0xf9, 0xef, 0xf5, 0xf2,
0xa0, 0xe6, 0xec, 0xe1, 0xe7, 0xa0, 0xf7, 0xe9, 0xf4, 0xe8, 0xa0, 0xf9, 0xef, 0xf5, 0xba, 0xa0,
0xec, 0xe1, 0xf3, 0xe1, 0xe3, 0xf4, 0xe6, 0xfb, 0xf4, 0xe8, 0xb3, 0xdf, 0xf3, 0xb0, 0xf5, 0xee,
0xe4, 0xf3, 0xdf, 0xb0, 0xe6, 0xdf, 0xf4, 0xb3, 0xf8, 0xf4, 0xfd
]


for i in s:
    print chr(i - 0x80),
```

The result:

```
# ./solve.py
I   s p e n t   s o   m u c h   t i m e   m a k i n g   t h i s   s o n g   f o r   y o u !   I   h o p e   y o u   l i k e   i t !   W h a t ?   Y o u   d o n ' t ?   I f   y o u   d o n ' t   a p p r e c i a t e   t h e   q u a l i t y   o f   m y   m u s i c   t h e n   y o u   c a n   j u s t   l e a v e .   A n d   d o n ' t   f o r g e t   t o   t a k e   y o u r   f l a g   w i t h   y o u :   l a s a c t f { t h 3 _ s 0 u n d s _ 0 f _ t 3 x t }
```

The flag was `lasactf{th3_s0unds_0f_t3xt}`
