### Solved by barrebas

And the Prophet Said was a 100 point crypto challenge.

<!--more-->

The download contains a text file with base64-encoded data, which becomes a .gz archive. After decompressing, I obtained a text file with biblical text. Not my cup of tea, but I immediately saw that certain sentences were duplicated. I wrote a python script to count the occurences of lines:


```python
d = {}

with open('text-file') as f:
    lines = f.readlines()
    for l in lines:
        if l in d:
            d[l] += 1
        else:
            d[l] = 1

for w in sorted(d, key=d.get):
    print w, d[w]
```

Which gave me these frequencies:


```bash
...snip...
Ye shall not eat anything with the blood: neither shall ye use enchantments, nor practise augury.
21
And ye shall keep my statutes, and do them: I am Jehovah who sanctifieth you.
23
And if a man lie with a beast, he shall surely be put to death: and ye shall slay the beast.
26
And when he hath made an end of atoning for the holy place, and the tent of meeting, and the altar, he shall present the live goat:
47
```

A total of 30 distinct strings were found. I guessed these strings represented letters, so I extended the python script a bit and started puzzling:

```python
d = {}

with open('text-file') as f:
    lines = f.readlines()
    for l in lines:
        if l in d:
            d[l] += 1
        else:
            d[l] = 1

for w in sorted(d, key=d.get):
    print w, d[w]

print len(d)    
x = dict(zip(sorted(d, key=d.get, reverse=True), ' etaoinsrhldcubkfgjmpqvwxyz012'))

# transpose the strings to letters and print out the message
out = ""
for l in lines:
    out += x[l]
    
print out
```

I guessed that the most common string was a space, which indeed yielded word- and sentence-like output:

```
rtiisq eroa grniitlht oa wtdc tnac oalje oe1 ojk edcolh es bdoet n islh ktaanht as aeneoaeoga boii rtip csf boer ert iteetd mdtxftlgotav ufe ojk lse dtniic hssy ne ernev s0 erneja tlsfhrq nrnrq ojk ol iswt boer aokpit isbtdgnat minha boersfe apngta nly aednlht ackusiav minhziureuhhfwmacleuuxbus2
```

I then started to look for THE, THIS, A, AND to guess the first couple of letters:

```
Heoosq THIS gHnooelre IS wedc enSc ISljT IT1 Ijk TdcIlr Ts bdITe n oslr keSSnre Ss STnTISTIgS bIoo Heop csf bITH THe oeTTed mdexfelgIeSv ufT Ijk lsT denooc rssy nT THnTv s0 THnTjS elsfrHq nHnHq Ijk Il oswe bITH SIkpoe osbedgnSe monrS bITHsfT SpngeS nly STdnlre SckusoSv monrzouHTurrfwmSclTuuxbus2
```

Slowly but surely, I translated all the letters, and the words slowly emerged (I love that!):

```
HEoonq THIS gHAooEldE IS VERY EASY ISljT IT1 Ijk TRYIld Tn WRITE A onld kESSAdE Sn STATISTIgS WIoo HEop Ynf WITH THE oETTER mRExfElgIESb ufT Ijk lnT REAooY dnnc AT THATb n0 THATjS ElnfdHq AHAHq Ijk Il onVE WITH SIkpoE onWERgASE moAdS WITHnfT SpAgES Alc STRAldE SYkunoSb moAdzouHTuddfVmSYlTuuxWun2

HELLoq THIS CHALLENGE IS VERY EASY ISN'T IT? I'k TRYING To WRITE A LoNG kESSAGE So STATISTICS WILL HELp Yof WITH THE LETTER mRExfENCIESb ufT I'k NoT REALLY Good AT THATb oj THAT'S ENofGHq AHAHq I'k IN LoVE WITH SIkpLE LoWERCASE mLAGS WITHofT SpACES ANd STRANGE SYkuoLSb mLAGzLuHTuGGfVmSYNTuuxWuo2

HELLOq THIS CHALLENGE IS VERY EASY ISN'T IT? I'M TRYING TO WRITE A LONG MESSAGE SO STATISTICS WILL HELP YOU WITH THE LETTER kRExUENCIESb uUT I'M NOT REALLY GOOd AT THATb Oj THAT'S ENOUGHq AHAHq I'M IN LOVE WITH SIMPLE LOWERCASE kLAGS WITHOUT SPACES ANd STRANGE SYMuOLSb kLAGzLuHTuGGUVkSYNTuuxWuO2

HELLO! THIS CHALLENGE IS VERY EASY ISN'T IT? I'M TRYING TO WRITE A LONG MESSAGE SO STATISTICS WILL HELP YOU WITH THE LETTER FREQUENCIES, BUT I'M NOT REALLY GOOD AT THAT, OK THAT'S ENOUGH! AHAH! I'M IN LOVE WITH SIMPLE LOWERCASE FLAGS WITHOUT SPACES AND STRANGE SYMBOLS, FLAG{LBHTBGGUVFSYNTBBQWBO}
```

The final translation dictionary was: ```x = dict(zip(sorted(d, key=d.get, reverse=True), " TESILAOHGNRYBWMUC'FP!,VQD{K?}"))```
The challenge already said that the flag needed a bit more work. Indeed, `flag{lbhtbgguvfsyntbbqwbo}` was not accepted. What then? Bitvijays suggested that the flag was another "ciphertext", so I thought of Caesar cipher. The easiest is [rot13](http://rot13.com) and indeed, the flag was `flag{yougotthisflagoodjob}`.

