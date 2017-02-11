AlexCTF Crypto 1: Ultracoded
---

### Solved by rasta_mouse

We're provided a file called `zero_one` with the following clue:

```Fady didn't understand well the difference between encryption and encoding, so instead of encrypting some secret message to pass to his friend, he encoded it!
Hint: Fady's encoding doens't handly any special character
```

```
➜  ultracoded file zero_one 
zero_one: ASCII text, with very long lines
```
```
➜  ultracoded cat zero_one 
ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ZERO ZERO ZERO ONE ZERO ONE ONE ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ONE ZERO ONE ZERO ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ONE ZERO ONE ZERO ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ZERO ZERO ONE ONE ZERO ZERO ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ZERO ZERO ZERO ONE ZERO ONE ZERO ZERO ZERO ONE ZERO ZERO ONE ONE ONE ONE ZERO ONE ZERO ZERO ONE ONE ONE ONE ZERO ONE
```

Convert the words `ZERO` and `ONE` to their binary equivalent and remove the spaces.

```
➜  ultracoded sed 's/ZERO/0/g; s/ONE/1/g; s/ //g' zero_one 
0100110001101001001100000110011101001100011010010011000001110101010011000110100101000001011101010100100101000011001100000111010101001100011010010011000001100111010011000101001100110100011101000100110001101001010000010111010001001001010000110011010001110101010011000101001100110100011001110100110001010011010000010111010101001100011010010011010001110101010010010100001100110100011101000100110001010011001100000111010001001001010000110011010001110101010011000110100100110100011101010100100101000011001100000111010001001100010100110100000101110101010011000101001100110000011101000100110001010011010000010111010101001100011010010011010001100111010011000101001100110000011101000100100101000011001101000111010101001100011010010011010001110101010010010100001100110100011101010100110001010011010000010111010101001100010100110011000001110101010010010100001100110100011101010100110001101001001100000111010001001001010000110011010001110100010011000110100101000001011101000100110001010011001100000110011101001100011010010011010001110101010011000110100100110100011001110100110001101001010000010111010001001100011010010011000001110101010010010100001100110100011101000100110001101001010000010111010101001100011010010011010001110100010011000101001101000001011101000100100101000011001100000111010001001100010100110100000101110100010010010100001100110000011101010100110001101001001100000110011101001100010100010011110100111101
```

Convert this binary into ASCII.

```
>>> import binascii
>>> n = int('0b0100110001101001001100000110011101001100011010010011000001110101010011000110100101000001011101010100100101000011001100000111010101001100011010010011000001100111010011000101001100110100011101000100110001101001010000010111010001001001010000110011010001110101010011000101001100110100011001110100110001010011010000010111010101001100011010010011010001110101010010010100001100110100011101000100110001010011001100000111010001001001010000110011010001110101010011000110100100110100011101010100100101000011001100000111010001001100010100110100000101110101010011000101001100110000011101000100110001010011010000010111010101001100011010010011010001100111010011000101001100110000011101000100100101000011001101000111010101001100011010010011010001110101010010010100001100110100011101010100110001010011010000010111010101001100010100110011000001110101010010010100001100110100011101010100110001101001001100000111010001001001010000110011010001110100010011000110100101000001011101000100110001010011001100000110011101001100011010010011010001110101010011000110100100110100011001110100110001101001010000010111010001001100011010010011000001110101010010010100001100110100011101000100110001101001010000010111010101001100011010010011010001110100010011000101001101000001011101000100100101000011001100000111010001001100010100110100000101110100010010010100001100110000011101010100110001101001001100000110011101001100010100010011110100111101', 2)
>>> binascii.unhexlify('%x' % n)
'Li0gLi0uLiAuIC0uLi0gLS4tLiAtIC4uLS4gLSAuLi4uIC4tLS0tIC4uLi4uIC0tLSAuLS0tLSAuLi4gLS0tIC4uLi4uIC4uLSAuLS0uIC4uLi0tIC4tLiAtLS0gLi4uLi4gLiAtLi0uIC4tLiAuLi4tLSAtIC0tLSAtIC0uLi0gLQ=='
```

Base64 this string.

```
➜  ultracoded echo -en Li0gLi0uLiAuIC0uLi0gLS4tLiAtIC4uLS4gLSAuLi4uIC4tLS0tIC4uLi4uIC0tLSAuLS0tLSAuLi4gLS0tIC4uLi4uIC4uLSAuLS0uIC4uLi0tIC4tLiAtLS0gLi4uLi4gLiAtLi0uIC4tLiAuLi4tLSAtIC0tLSAtIC0uLi0gLQ== | base64 -d
.- .-.. . -..- -.-. - ..-. - .... .---- ..... --- .---- ... --- ..... ..- .--. ...-- .-. --- ..... . -.-. .-. ...-- - --- - -..- -
```

I used [this website](http://morsecode.scphillips.com/translator.html) to translate this Morse Code into text.

```
ALEXCTFTH15O1SO5UP3RO5ECR3TOTXT
```

Finally, substitute the `O`'s for `_`'s and plop in the curly braces.

```
ALEXCTF{TH15_1S_5UP3R_5ECR3T_TXT}
```