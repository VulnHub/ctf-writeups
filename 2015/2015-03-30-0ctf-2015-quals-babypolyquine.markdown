### Solved by superkojiman

This was a 100 point challenge where we had to submit a quine that worked in at least three of five languages. The quine was tested against C, Python 2, Python 3, Ruby, and Perl.

When I connected to the server, I was asked to input the source code followed by a null byte to terminate it. Since an empty source file is sort of a quine, sending just a null byte gives passes for four out of five of the languages, and rewards us with the flag: 

```
$ nc 202.112.26.114 12321
Welcome to 0ops Polyglot Quine Challenge!
Please send the source code, terminating with null byte. The maximum allowed length is 1200 bytes.
Current Shortest polyquine is 416 bytes.
^@
Your source code length is 0 bytes.
Running...
Results:
C: Incorrect
Python2: Correct
Python3: Correct
Ruby: Correct
Perl: Correct

Now you are a polyglot quine beginner.
Here is a little reward: 0ctf{The very moment of raising beginner's mind is the accomplishment of true awakening itself}
Try to get all 5 correct next time.
```

Flag: **0ctf{The very moment of raising beginner's mind is the accomplishment of true awakening itself}**
