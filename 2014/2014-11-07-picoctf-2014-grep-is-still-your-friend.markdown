### Solved by bitvijays

Grep is your friend is a 40 point forensics challenge. You are required to search a key for a particular file.

> The police need help decrypting one of your father's files. Fortunately you know where he wrote down all his backup decryption keys as a backup (probably not the best security practice). You are looking for the key corresponding to daedaluscorp.txt.enc. The file is stored on the shell server at /problems/grepfriend/keys.

For this challenge, we need to search for the daedaluscorp.txt.enc inside directories or files. Grep allows you to search inside the files too. 
```
grep -rnw -e "daedaluscorp.txt.enc" 
grep -rnw '/problems/grepfriend/keys' -e "daedaluscorp.txt.enc"
-r                    : search recursively
-n                    : print line number
-w                    : match the whole word.
-e                    : pattern to search for


pico*****@shell:~$ grep -rnw '/problems/grepfriend/keys' -e "daedaluscorp.txt.enc"
5865:daedaluscorp.txt.enc	b2bee8664b754d0c85c4c0303134bca6

```

The flag is **b2bee8664b754d0c85c4c0303134bca6**

