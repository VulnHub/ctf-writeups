AlexCTF crypto2
---

### Solved by barrebas

For crypto2, we are given an encrypted message. According to the challenge, the message is encrypted with a one time pad, which should be unbreakable. The one time pad, however, has been re-used, allowing us to recover the plaintext by using a technique known as ```cribdragging```.

There is a nice [https://github.com/SpiderLabs/cribdrag](tool) to aid us. The first part of the key is guessed to be ```ALEXCTF{```, as it makes sense to assume that the one time pad is the flag.

We feed the message into the cribdrag script.

```
user@lithium:~/alexctf/cr2$ python ./cribdrag/cribdrag.py `cat ./msg |tr -d '\n'`
Your message is currently:
0	________________________________________
40	________________________________________
80	________________________________________
120	________________________________________
160	________________________________________
200	________________________________________
240	________________________________________
280	____
Your key is currently:
0	________________________________________
40	________________________________________
80	________________________________________
120	________________________________________
160	________________________________________
200	________________________________________
240	________________________________________
280	____
Please enter your crib: ALEXCTF{
*** 0: "Dear Fri"
1: "hho;Q`TV"
2: "ef&JwFkP"
3: "k/WlQymM"
...
Enter the correct position, 'none' for no match, or 'end' to quit: 0
Is this crib part of the message or key? Please enter 'message' or 'key': key
Your message is currently:
0	Dear Fri________________________________
40	________________________________________
80	________________________________________
120	________________________________________
160	________________________________________
200	________________________________________
240	________________________________________
280	____
Your key is currently:
0	ALEXCTF{________________________________
40	________________________________________
80	________________________________________
120	________________________________________
160	________________________________________
200	________________________________________
240	________________________________________
280	____
Please enter your crib: 
```


This was a start, but not super helpful. I assumed the key len was the length of a line, which amounted to 26 bytes. I wrote a small helper script:

```python
with open('msg') as f:
	data = f.readlines()
	f.close()

crypt = ''
for line in data:
	crypt += line.strip().decode('hex')
#user@lithium:~/alexctf/cr2$ #used cribdrag.py 

key = 'ALEXCTF{'
key +=  ' ' * (26 - len(key))
decoded = ''
for i in range(len(crypt)):
	decoded += chr(ord(key[i % len(key)]) ^ ord(crypt[i]))
print decoded
```

This script yielded the following output (piped through strings):

```bash
Dear Fri
K,Y(nderstoo
Y(sed One 
2n schemeDE;E
}is the o
)hod thatH
$ proven 
}ever if 
8cure, Le
Y<gree wit
Y8ncryptio
```

Now we return to cribdrag.py, and we start guessing words of the plaintext. 

```bash
Please enter your crib: used One Time Pad
...
51: "}ALEXCTF{hERE_gOE"
```

Among the lines of garbled text is this piece. It contains something that very well might be part of the key! Updating ```decode.py``` with ```ALEXCTF{hERE_gOES``` (the last S was guessed) and re-running it yields

```bash
Dear FriEnd, thi
K,Y(nderstooD my Mis
Y(sed One Time PadS
2n scheme
 I hEar
}is the oNly eNcr
)hod that
is mAth
$ proven To be
}ever if The kEy 
8cure, LeT Me Kno
Y<gree witH me To 
Y8ncryptioN schEmeS
```

Again, we return to the cribdrag, now guessing "is mathematically proven to be" as the crib. This yields, besides plenty of garbage, ```139: "ERE_GOES_THE_KEY}ALEXCTF{HERE_"``` so the key could very well be ```ALEXCTF{HERE_GOES_THE_KEY}```. This is also the correct flag!
