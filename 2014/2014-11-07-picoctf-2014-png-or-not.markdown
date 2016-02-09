### Solved by bitvijays

PNG or Not is a 100 point forensics challenge. You are provided with a image.png file.

> On a corner of the bookshelf, you find a small CD with an image file on it. It seems that this file is more than it appears, and some data has been hidden within. Can you find the hidden data?

First step is always confirming with the file command.

```
file image.png 
image.png: PNG image data, 280 x 280, 8-bit/color RGB, non-interlaced
```
file command confirms that it's PNG file. Next step was to view the file for hidden data using hexdump. PNG file contains IHDR and IEND which denotes the start and end of the PNG file. viewing the file in hexdump confirms this. It also shows that after IEND. There is data present with 7z and ascii representing flag.txt.
```
hexdump -C image.png

00000000  89 50 4e 47 0d 0a 1a 0a  00 00 00 0d 49 48 44 52  |.PNG........IHDR|
00000010  00 00 01 18 00 00 01 18  08 02 00 00 00 08 ec 7e  |...............~|
00000020  db 00 00 00 06 74 52 4e  53 00 fe 00 01 00 fd 5b  |.....tRNS......[|
00000030  6c 0d 3b 00 00 05 c1 49  44 41 54 78 9c ed dd 41  |l.;....IDATx...A|
*************Trimmed************
000005f0  02 42 82 80 90 20 f0 0b  3d 2b 08 3b 89 af 07 b9  |.B... ..=+.;....|
00000600  00 00 00 00 49 45 4e 44  ae 42 60 82 37 7a bc af  |....IEND.B`.7z..|
00000610  27 1c 00 03 b8 64 d3 c1  1a 00 00 00 00 00 00 00  |'....d..........|
00000620  50 00 00 00 00 00 00 00  b5 6b 69 46 00 22 92 c6  |P........kiF."..|
00000630  ae 77 46 b4 23 6d f7 5d  c0 c0 a4 dc 1f a8 38 05  |.wF.#m.]......8.|
00000640  57 b9 76 3e 20 00 01 04  06 00 01 09 1a 00 07 0b  |W.v> ...........|
00000650  01 00 01 23 03 01 01 05  5d 00 00 01 00 0c 14 00  |...#....].......|
00000660  08 0a 01 dc e1 0d de 00  00 05 01 11 13 00 66 00  |..............f.|
00000670  6c 00 61 00 67 00 2e 00  74 00 78 00 74 00 00 00  |l.a.g...t.x.t...|
00000680  14 0a 01 00 90 d6 20 07  48 db cf 01 15 06 01 00  |...... .H.......|
00000690  20 00 00 00 00 00                                 | .....|
00000696
```

7z represents a archive and file can be extracted using 7z.
```
7z x image.png 

7-Zip [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
p7zip Version 9.20 (locale=en_GB.utf8,Utf16=on,HugeFiles=on,4 CPUs)

Processing archive: image.png

Extracting  flag.txt

Everything is Ok

Size:       20
Compressed: 1686

cat flag.txt
EKSi7MktjOpvwesurw0v
```
The flag is **EKSi7MktjOpvwesurw0v**

