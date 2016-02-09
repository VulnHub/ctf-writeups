### Solved by bitvijays

Ceasar is a 20 point cryptography challenge. This is another one of the easy ones. 

> You find an encrypted message written on the documents. Can you decrypt it?

The encrypted text in question looks like this:

```
uiftfdsfuqbttqisbtfjtpgtqyrdhekuqsxjdtvyvkghlpvkfml
```

We know it's a ceasar cipher based on the challenge's name. The fastest way to solve it was to just put in [http://planetcalc.com/1434/](http://planetcalc.com/1434/). It presents you all the possible values of the plaintext based on ROT [0-25]. ROT25 presents you with thesecretpassphraseisofspxqcgdjtprwicsuxujfgkoujelk.
<br><br>Another possible way of solving this is use <a href="https://www.cryptool.org/en/cryptool1">Cryptool1</a> software, which provides the analysis of Symmetric Key Encrption, just paste the encryption text in the Window. Go to Analysis > Symmetric Encryption (Classic) > Ciphertext Only > Caesar.

It solves it instantly. The flag is **ofspxqcgdjtprwicsuxujfgkoujelk**

