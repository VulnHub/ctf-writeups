###Solved by bitvijays

Locate the flag in all the words at /problems/grep-quest_0/grepy-words/

We just needed to login to the ssh server and in the folder /problems/grep-quest_0/grepy-words/ multiple files were present and do a recursive grep search for lasa

```
/problems/grep-quest_0/grepy-words$ grep -r lasa
potato.txt:lasactf{1_am_a_h1dd3n_p0tat0}
```

The flag is **lasactf{1_am_a_h1dd3n_p0tat0}**

