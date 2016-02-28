### Solved by superkojiman and barrebas

This was a miscellaneous challenge titled "K". A text file containing the following was provided:

```
. . . . . . . . . . . . . . .
. . . . . ! ? ! ! . ? . . . .
. . . . . . . . . . . . . . .
. ? . ? ! . ? . . . . . . . .
! . ? . . . . . . . ! ? ! ! .
? . . . . . . ? . ? ! . ? . .
. . . . . . . . ! . ! ! ! ! !
! ! . . . ! . . . . . . . . .
. . . . ! . ? . . . . . . . !
? ! ! . ? ! ! ! ! ! ! ? . ? !
. ? ! ! ! ! ! . . . . . . . .
. . . . . ! . . . . . ! . ? .
. . . . . . . . ! ? ! ! . ? !
! ! ! ! ! ! ! ? . ? ! . ? ! .
! ! ! ! ! ! ! ! ! . ! . ? . .
. . . . . . . ! ? ! ! . ? . .
. . . . . . ? . ? ! . ? . . .
. . . . . . . . . ! . ! ! ! !
! . ? . . . . . . . . . ! ? !
! . ? ! ! ! ! ! ! ! ! ? . ? !
. ? ! . ! ! ! ! ! ! ! ! ! ! !
. . . ! . . . . . . . . . . .
! . ! ! ! . ! ! ! ! ! ! ! ! !
. ? . . . . . . . . . ! ? ! !
. ? . . . . . . . . ? . ? ! .
? ! . ! ! ! ! ! ! ! ! ! . ! !
! ! ! ! ! ! ! ! ! ! ! ! ! ! !
. . . . . . . . . . . . . ! .
? . 
```

I figured this was some kind of esoteric programming language, and after a bit of Googling, I realized it was a short form of OOK. I ran it through an online OOK interpreter I found, and it returned the following:

```
hvstzouwgccywgbchgcsogm
```

That wasn't the flag. barrebas mentioned that it might be encrypted with a rotation cipher. He was right, turns out it was ROT-12 encoded. The resulting flag was ```theflagisookisnotsoeasy```. 
