### Solved by superkojiman

Substitution is a 50 point cryptography challenge. This is another one of the easy ones. 

> There's an authorization code for some Thyrin Labs information here, along with someone's favorite song. But it's been encrypted! Find the authorization code.

The encrypted text in question looks like this:

```
tep yhteszxdytxsj rsbp xo yuesgpjpuuszgb

x ryj oesu fsh tep uszgb
oexjxjk oexccpzxjk ongpjbxb
tpgg cp nzxjrpoo jsu uepj bxb
fsh gyot gpt fshz epyzt bprxbp

x ryj snpj fshz pfpo
tyap fsh usjbpz qf usjbpz
slpz oxbpuyfo yjb hjbpz
sj y cykxr ryznpt zxbp

y uesgp jpu uszgb
y jpu iyjtyotxr nsxjt si lxpu
js sjp ts tpgg ho js
sz uepzp ts ks
sz oyf upzp sjgf bzpycxjk

y uesgp jpu uszgb
y byddgxjk ngyrp x jplpz ajpu
qht jsu izsc uyf hn epzp
xto rzfotyg rgpyz
teyt jsu xc xj y uesgp jpu uszgb uxte fsh

hjqpgxplyqgp oxketo
xjbporzxqyqgp ippgxjk
osyzxjk thcqgxjk izppueppgxjk
tezshke yj pjbgpoo bxycsjb oaf

y uesgp jpu uszgb
bsjt fsh byzp rgsop fshz pfpo
y ehjbzpb teshoyjb texjko ts opp
esgb fshz qzpyte  xt kpto qpttpz
xc gxap y oesstxjk otyz
xlp rscp os iyz
x ryjt ks qyra ts uepzp x hopb ts qp

y uesgp jpu uszgb
plpzf thzj y ohznzxop
uxte jpu eszxdsjo ts nhzohp
plpzf cscpjt kpto qpttpz
xgg reyop tepc yjfuepzp
tepzpo txcp ts onyzp
gpt cp oeyzp texo uesgp jpu uszgb uxte fsh

y uesgp jpu uszgb

y uesgp jpu uszgb
y jpu iyjtyotxr nsxjt si lxpu
js sjp ts tpgg ho js
sz uepzp ts ks
sz oyf upzp sjgf bzpycxjk
y uesgp jpu uszgb
plpzf thzj y ohznzxop
uxte jpu eszxdsjo ts nhzohp
plpzf cscpjt kpto qpttpz
xgg reyop tepc yjfuepzp
tepzpo txcp ts onyzp
xgg reyop tepc yjfuepzp
tepzpo txcp ts onyzp
gpt cp oeyzp texo uesgp jpu uszgb uxte fsh

y uesgp jpu uszgb
y uesgp jpu uszgb
teyto uepzp upgg qp
teyto uepzp upgg qp
y tezxggxjk reyop
y usjbzsho ngyrp
isz fsh yjb cp
```

We know it's a substitution cipher based on the challenge's name. The fastest way to solve it was to just plug in the first few lines in [http://quipqiup.com/index.php](http://quipqiup.com/index.php). I left the clues field blank, and set it to auto-detect the puzzle. 

![](/images/2014/pico/substitution/01.png)

It solves it instantly. The flag is **awholenewworld**

