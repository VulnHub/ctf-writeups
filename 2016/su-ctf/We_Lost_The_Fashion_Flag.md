###Solved by bitvijays

***Challenge 3:We lost the Fashion Flag!: 100 Points*** In Sharif CTF we have lots of task ready to use, so we stored their data about author or creation date and other related information in some files. But one of our staff used a method to store data efficiently and left the group some days ago. So if you want the flag for this task, you have to find it yourself.

```
tar -xzvf fashion.tar.gz
sharif_tasks.tgz
fashion.model
```
Running file on fashion.model reveals it's a data
```
file fashion.model 
fashion.model: data
```
Let's find out what it might contain by using string
```
strings fashion.model | more

FemtoZip
4}', 'ctf': 'Shairf CTF', 'points': 265,4}', 'ctf': 'Shairf CTF',
***SNIP***

#Extracting sharif_tasks.tgz which provides a folder full of multiple files

gunzip sharif_tasks.tgz
tar -xvf sharif_tasks.tar
```

Searching about Femtozip we reach <a href="https://github.com/gtoubassi/femtozip/wiki">FemtoZip</a> which is a "shared dictionary" compression library optimized for small documents that may not compress well with traditional tools such as gzip. 

Reading more about it in the tutorial, installing it and how to use it for decompressing.
```
fzip --model /tmp/data/model.fzm --decompress /tmp/data/benchmark

using it for our challenge

fzip --decompress --model ./fashion.model out/
```
This provides us with a lot of decompressed file containing
```
cat 314
{'category': 'web', 'author': 'staff_1', 'challenge': 'Fashion', 'flag': 'SharifCTF{6f7e9b3cfe0f5a9760b395c7a6e139fc}', 'ctf': 'Shairf CTF', 'points': 195, 'year': 2013}
cat 31
{'category': 'crypto', 'author': 'staff_4', 'challenge': 'Fashion', 'flag': 'SharifCTF{f31d7e0850b8217f64e8bcdc491581ee}', 'ctf': 'Shairf CTF', 'points': 275, 'year': 2012}
```
Greping for the year 2016, points 100, and category Forensics
```
for i in $(ls); do cat $i >> testfile; >> testfile; done;

cat testfile | grep 2016 | grep forensic | grep 100

{'category': 'forensic', 'author': 'staff_3', 'challenge': 'Fashion', 'flag': 'SharifCTF{2b9cb0a67a536ff9f455de0bd729cf57}', 'ctf': 'Shairf CTF', 'points': 100, 'year': 2016}
{'category': 'forensic', 'author': 'staff_5', 'challenge': 'Fashion', 'flag': 'SharifCTF{41160e78ad2413765021729165991b54}', 'ctf': 'Shairf CTF', 'points': 100, 'year': 2016}
{'category': 'forensic', 'author': 'staff_2', 'challenge': 'Fashion', 'flag': 'SharifCTF{8725330d5ffde9a7f452662365a042be}', 'ctf': 'Shairf CTF', 'points': 100, 'year': 2016}
{'category': 'forensic', 'author': 'staff_3', 'challenge': 'Fashion', 'flag': 'SharifCTF{1bc898076c940784eb329d9cd1082a6d}', 'ctf': 'Shairf CTF', 'points': 100, 'year': 2016}
{'category': 'forensic', 'author': 'staff_6', 'challenge': 'Fashion', 'flag': 'SharifCTF{2b10008f10c84f83f0ec1f08609947cf}', 'ctf': 'Shairf CTF', 'points': 160, 'year': 2016}
{'category': 'forensic', 'author': 'staff_5', 'challenge': 'Fashion', 'flag': 'SharifCTF{e11006dfebfe5d47efe15174fdc0929b}', 'ctf': 'Shairf CTF', 'points': 85, 'year': 2016}
{'category': 'forensic', 'author': 'staff_6', 'challenge': 'Fashion', 'flag': 'SharifCTF{c19285fd5d56c13b169857d863a1b437}', 'ctf': 'Shairf CTF', 'points': 100, 'year': 2016}
````

The flag is ***SharifCTF{2b9cb0a67a536ff9f455de0bd729cf57}***

