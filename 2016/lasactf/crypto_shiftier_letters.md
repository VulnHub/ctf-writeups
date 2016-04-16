### Solved by superkojiman


This was an interesting crypto challenge worth 120 points. Connecting to the server would return a random string that was encrypted with ROT 0-25. I needed to figure out what value was used to shift each letter in the string, and then send back the decrypted message. If I sent the correct response, I'd receive another random encrypted string with a different value used to shift each letter. Send the correct decrypted response, and it starts all over again for a total of 50 iterations. 

I used [http://planetcalc.com/1434/](http://planetcalc.com/1434/) to quickly decrypt each string the server sent, and based on what I saw, some of the words in the decrypted sentence seemed nonsensical, and wouldn't exist in /usr/share/dict/words. However, at least one or two of them did, so that meant I would need to check each encrypted word sent by the server to see if I could find a matching word in my dictionary. 

To speed things up, I decided to read in /usr/share/dict/words, encrypt each line with ROT 0-25, and store the encrypted word as a key in a Python dictionary, with the number used to shift the letter as the value. That is:

```
encrypted_word => rot_value
```

Once I had a dictionary built, it would be easy to check each encrypted word sent by the server against my dictionary, and if I found a match, I could get the number used to shift each letter, and therefore decrypt the entire string. 

Here's the Python code I came up with: 

```python
#!/usr/bin/env python

from pycipher import *
from pwn import *

# build a dictionary of encrypted words as the key, and the value 
# as the key they were rotated with
# encrypted -> rot_value

rot_dict = {}

print "Building dictionary..."

for word in open("words.txt"):
    word_stripped = word.rstrip()
    for i in range(0, 26):
        encrypted = Caesar(i).encipher(word_stripped).lower()
        rot_dict[encrypted] = i

print "Done!"


# connect to server
r = remote("web.lasactf.com", 4056)

# receive from server until we get a flag or an error
problem_num = 0
while True:
    problem_num += 1

    d = r.recv().rstrip()
    print "\nProblem %d: Received: %s" % (problem_num, d)

    if "lasactf" in d:
        print "We're done! Flag was returned!"
        break

    if "Incorrect" in d:
        print "Something went wrong :("
        break

    plaintext = ""
    rot_value = 0
    found_key = False
    for word in d.split():
        key = word.rstrip()

        "Looking for", key
        if key in rot_dict:
            rot_value = rot_dict[key]
            found_key = True
            print "Found ROT value", rot_value
            print "%s ROT %d = %s" % (key, rot_value, Caesar(rot_value).decipher(key).lower())
            break

    if found_key:
        print "Decrypting message..."
        for word in d.split():
            word_stripped = word.rstrip()
            plaintext += Caesar(rot_value).decipher(word_stripped).lower() + " "
    else:
        # assume not encrypted, and just send the plaintext right back
        print "Nothing to decrypt, or couldn't find any keys"

    print "Sending plaintext [%s]" % (plaintext.rstrip(),)
    r.sendline(plaintext.rstrip())
```

And here we go:

```
# ./shiftier.py
Building dictionary...
Done!
[+] Opening connection to web.lasactf.com on port 4056: Done

Problem 1: Received: ufodho gwozwo dobofwhwia rsbrfczcuwgh aoqfcamszcb hcczghcqy szsasbhozwga qoggccb hmdvoqsos qcbgsbhwsbh fsjszohwcbsf hgvw awgqcadzowb dofottwbwns aofhmfczcuwqoz hfwhvwcqofpcbohs bcbbobh gibrsfkwgs rworcqvwhs hvmasbs qifghzm vcdwbuzm psuons uibbsfgvwd
Found ROT value 14
ufodho ROT 14 = grapta
Decrypting message...
Sending plaintext [grapta sialia panaritium dendrologist macromyelon toolstock elementalism cassoon typhaceae consentient revelationer tshi miscomplain paraffinize martyrological trithiocarbonate nonnant sunderwise diadochite thymene curstly hopingly begaze gunnership]

Problem 2: Received: bzqgyapqdym aebtkmxsum oaynruet bmybtxqfqd exgnnk bdmfukmemygfbmpm bdqoaybxqfqzqee wzaffuzs baefbmxgpmx rxuzf yakqzzq fabafkbq yuodaequeyuo fqzmzfetub zazmeeqzfqp mxabqouef fajazaeue pgfoturk rmfndmuzqp mefdabtafasdmbtuo exmhazuomxxk tkndupmx emxuomoqage nubaxmdufk kffdarxgadufq
Found ROT value 12
bzqgyapqdym ROT 12 = pneumoderma
Decrypting message...
Sending plaintext [pneumoderma osphyalgia combfish pamphleter slubby pratiyasamutpada precompleteness knotting postpaludal flint moyenne topotype microseismic tenantship nonassented alopecist toxonosis dutchify fatbrained astrophotographic slavonically hybridal salicaceous bipolarity yttrofluorite]
.
.
.
Problem 50: Received: umspqbizqab xtibgkzivqi aivb emiaivl awixewwl cvwjabzckbmlvmaa kwvditmakmvbtg mfbziagttwoqabqk jwzwcopapqx pmuwotwjqvczqi qbmuqhmz jtizvql bzqxxmz gikpbuiv xgzwxcvkbczm illi vwvkwpmzmvb zmquxwam kikwkvmuqi qvitqmvijqtqbg kgxzqvwlwvbqlim vwdmulqoqbibm
Found ROT value 8
umspqbizqab ROT 8 = mekhitarist
Decrypting message...
Sending plaintext [mekhitarist platycrania sant weasand soapwood unobstructedness convalescently extrasyllogistic boroughship hemoglobinuria itemizer blarnid tripper yachtman pyropuncture adda noncoherent reimpose cacocnemia inalienability cyprinodontidae novemdigitate]

Problem 51: Received: You made it to the end! lasactf{shif73d-3n0ugh-ar3-we}
We're done! Flag was returned!
[*] Closed connection to web.lasactf.com port 4056
```

Building the dictionary took about 4 minutes, but once that was done, it was smooth sailing all the way to the end. After 50 iterations, the server returned the flag: `lasactf{shif73d-3n0ugh-ar3-we}`
