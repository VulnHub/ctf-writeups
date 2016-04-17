### Solved by z0rex

"Wher iz mai cheezburgr" was a forensics challenge worth 80 points.

> Halp! I lost my cheezburger and I can't find it! It's in this file somewere,
> can you be finds it? K Thx m8 much appreciat. 

I downloaded the attached `.dmg` file and mounted it using `hfsplus`

```
# mount -t hfsplus -o loop meow.dmg /mnt/tmp
```

Next I had a look at the files and folders

```
# ls -la 
total 532
drwxrwxr-x 1 root root     10 Dec  7 02:02 .
drwxr-xr-x 3 root root   4096 Apr 10 08:48 ..
-rw-r--r-- 1  501 utmp   8196 Jan  1 17:10 .DS_Store
dr-xr-xr-t 1 root root      2 Dec  7 01:29 '.HFS+ Private Directory Data'$'\r'
d-wx-wx-wt 1  501 utmp      3 Dec  7 14:09 .Trashes
drwx------ 1  501 utmp     11 Jan  1 17:10 .fseventsd
---------- 1 root root 524288 Dec  7 01:29 .journal
---------- 1 root root   4096 Dec  7 01:29 .journal_info_block
drwxrwxrwx 1  501 utmp     56 Jan  1 17:10 chezbrgeur
```


I spent a decent amount of time trying to find the solution in the folder 
`chezbrgeur`, but it was a dead end... So what about the folder `.Trashes`?
Folders with deleted files tends to have some interesting things in them :)

```
# cd .Trashes
> .Trashes # ls -la 
total 0
d-wx-wx-wt 1  501 utmp  3 Dec  7 14:09 .
drwxrwxr-x 1 root root 10 Dec  7 02:02 ..
drwx------ 1  501 utmp  6 Jan  1 17:05 501
> .Trashes # cd 501
> 501 # ls -la 
total 20
drwx------ 1 501 utmp    6 Jan  1 17:05 .
d-wx-wx-wt 1 501 utmp    3 Dec  7 14:09 ..
-rw-r--r-- 1 501 utmp 6148 Jan  1 17:05 .DS_Store
-rwxrwxrwx 1 501 utmp 2555 Dec  7 01:49 mysteries.fs
-rwxrwxrwx 1 501 utmp  885 Dec  7 14:09 'secret 12.05.37.fs'
-rwxrwxrwx 1 501 utmp  885 Dec  7 01:56 secret.fs
```

Secrets and mysteries are some interesting file names indeed. Let's take a closer
look.


```
# file mysteries.fs 
mysteries.fs: gzip compressed data, last modified: Mon Dec  7 01:49:42 2015, from Unix
# file secret\ 12.05.37.fs 
secret 12.05.37.fs: Zip archive data, at least v1.0 to extract
# file secret.fs 
secret.fs: Zip archive data, at least v1.0 to extract
```

So we have one gzip and two zip files. Let's copy them and see what we can find.

I started by renaming mysteries giving it the `.gz` extension and extracted it
with `gunzip`

```
# mv mysteries.{fs,gz}
# gunzip mysteries.gz 
# file mysteries
mysteries: POSIX tar archive
```

Ok... So now it's a tar file. Let's extract that as well

```
# tar xvf mysteries
./._log.txt
log.txt
my_passwords.md
```

That was almost too easy, so I was almost certain this was just another evil
troll. I checked the file `my_passwords.txt`, but it has so much text in it that
I decided to check `log.txt` first.

```
# cat log.txt 
zip -er secret.zip SECRETDONOTTOUCH/
me0wn3k0i<3c4tt3sv3rymch
me0wn3k0i<3c4tt3sv3rymch
exit
```

This is almost too good to be true! Let's try extracting `secrest.fs` with that
password.

```
# unzip -P 'me0wn3k0i<3c4tt3sv3rymch' secret.fs 
Archive:  secret.fs
   creating: SECRETSDONOTTOUCH/
  inflating: SECRETSDONOTTOUCH/._flag.txt  
 extracting: SECRETSDONOTTOUCH/flag.txt  
# cat SECRETSDONOTTOUCH/flag.txt 
angstromCTF{120x29y_dmg_no_haz_mai_chezburger}
```

Success! 

Flag: `angstromCTF{120x29y_dmg_no_haz_mai_chezburger}`