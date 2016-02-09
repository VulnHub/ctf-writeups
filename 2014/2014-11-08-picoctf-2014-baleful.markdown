### Solved by barrebas and superkojiman

#### Solving it with Pin

superkojiman here. I started off doing the usual analysis with gdb, IDA, and Hopper on the binary. barrebas was working on the same challenge as well, and while I was trying to make sense of the massive jump table at 0x0804898B, barrebas sent me a link to post by Jonathan Salwan titled [A binary analysis, count me if you can](http://shell-storm.org/blog/A-binary-analysis-count-me-if-you-can/). The post describes using a a tool called [Pin](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool) to count the number of instructions executed by a binary to guess the password. Much like baleful, Jonathon used it to solve a crackme, so we thought, why not give it a shot. 

I downloaded pin-2.12-58423-gcc.4.4.7-linux.tar.gz, unpacked it and compiled it. 

```
# tar xvzf pin-2.12-58423-gcc.4.4.7-linux.tar.gz
# cd pin-2.12-58423-gcc.4.4.7-linux/source/tools
make
```

It should fail with an error regarding Tsx:

```
make -k -C Tsx
make: Entering an unknown directory
make: *** Tsx: No such file or directory.  Stop.
make: Leaving an unknown directory
make[1]: [Tsx.build] Error 2 (ignored)
make[1]: Leaving directory `/root/baleful/pin-2.12-58423-gcc.4.4.7-linux/source/tools'
```

Not a problem, we only need source/tools/ManualExamples/obj-ia32/inscount0.so built, and it should have been built properly. Finally, copied the unpacked baleful into pin-2.12-58423-gcc.4.4.7-linux 

So the first thing that needs to be done is to determine how many characters the password is. From looking at the start of the jump table in IDA, we can see that it expects to loop through the input 30 times, so we know it's 30 characters long. However if we didn't know this, we can assume that the check on the password wouldn't occur until the binary got the correct password length. We can guess the password length by using pin: 

```
# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "a" > /dev/null 2>&1; cat inscount.out 
Count 688964

# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "aa" > /dev/null 2>&1; cat inscount.out 
Count 689815

# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "aaa" > /dev/null 2>&1; cat inscount.out 
Count 690666
```

I ran it three times, and notice that the instruction count changes each time. There's a difference of 851 instructions between each run. The idea is to run it until the difference changes, which indicates that we've guessed the correct password length, and so the binary has started checking each character in the password to determine if it's correct. To simplify the procedure, I wrote a script: 


```bash
# ./bale_count.sh 
diff between a:1 and aa:2 = 851
diff between aa:2 and aaa:3 = 851
diff between aaa:3 and aaaa:4 = 851
diff between aaaa:4 and aaaaa:5 = 851
.
.
.
diff between aaaaaaaaaaaaaaaaaaaaaaaaaa:26 and aaaaaaaaaaaaaaaaaaaaaaaaaaa:27 = 851
diff between aaaaaaaaaaaaaaaaaaaaaaaaaaa:27 and aaaaaaaaaaaaaaaaaaaaaaaaaaaa:28 = 851
diff between aaaaaaaaaaaaaaaaaaaaaaaaaaaa:28 and aaaaaaaaaaaaaaaaaaaaaaaaaaaaa:29 = 851
diff between aaaaaaaaaaaaaaaaaaaaaaaaaaaaa:29 and aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:30 = 48459
aaaaaaaaaaaaaaaaaaaaaaaaaaaaa and aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa have different instruction count
```

Sure enough the difference between 29 characters and 30 characters is a whopping 48,459 indicating that more instructions were executed. Now that we have the password length, we can brute force the password character by character. The program exits when the incorrect character is guessed, so like before, we should get the same difference between each run when the password is incorrect. 

My initial tests had a typo which generated incorrect results, and so I dismissed it thinking that I couldn't apply the technique to this challenge. By the time I decided to take a look at it again, barrebas had managed to manually solve a good part of the password. It was at this point I noticed the typo and realized that it was possible to automate solving the rest of the challenge. 

Let's take a look at how this works:

```text
# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "a_____________________________" > /dev/null 2>&1 ; cat inscount.out 
Count 761215

# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "b_____________________________" > /dev/null 2>&1 ; cat inscount.out 
Count 761215

# ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "c_____________________________" > /dev/null 2>&1 ; cat inscount.out 
Count 761215
```

There's a difference of 0 between each run. We assume that the password must be within the printable ASCII range, so once again, I wrote a script to brute force each character: 

```bash
#!/bin/bash

expected=0
known=""

for j in `seq 0 30`; do 
    for i in `seq 33 127`; do
        guess1=`printf "\x$(printf %x $i)"`
        guess2=`printf "\x$(printf %x $(($i+1)))"`

        if [[ -z $known ]]; then 
            pass1="${guess1}_____________________________"
            pass2="${guess2}_____________________________"
        else
            pass1="${known}${guess1}"
            pass2="${known}${guess2}"

            padding=$((29-${#pass1}))
            for j in `seq 0 $padding`; do 
                pass1=${pass1}"_"
                pass2=${pass2}"_"
            done
        fi 

        ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "${pass1}" > /dev/null 2>&1
        count1=`cut -d' ' -f2 inscount.out | tr -d '\n'`

        ./pin -t source/tools/ManualExamples/obj-ia32/inscount0.so -- ./baleful <<< "${pass2}" > /dev/null 2>&1
        count2=`cut -d' ' -f2 inscount.out | tr -d '\n'`
        diff=$(($count2-$count1))
        echo "diff between $pass1:${#pass1} and $pass2:${#pass2} = $diff ; known: $known"

        if [[ $diff -ne $expected ]]; then
            echo "$pass1 and $pass2 have different instruction count: $diff"
            known=${known}${guess2}
            break
        fi

        pass1="${pass1}a"
        pass2="${pass2}a"
    done
done
```

Let's run it:

```text
# ./bale_brute.sh 
diff between !_____________________________:30 and "_____________________________:30 = 0 ; known: 
diff between "_____________________________:30 and #_____________________________:30 = 0 ; known: 
diff between #_____________________________:30 and $_____________________________:30 = 0 ; known: 
diff between $_____________________________:30 and %_____________________________:30 = 0 ; known: 
diff between %_____________________________:30 and &_____________________________:30 = 0 ; known: 
.
.
.
diff between n_____________________________:30 and o_____________________________:30 = 0 ; known: 
diff between o_____________________________:30 and p_____________________________:30 = -12 ; known: 
o_____________________________ and p_____________________________ have different instruction count: -12
diff between p!____________________________:30 and p"____________________________:30 = 0 ; known: p
diff between p"____________________________:30 and p#____________________________:30 = 0 ; known: p
.
.
.
diff between paa___________________________:30 and pab___________________________:30 = 0 ; known: pa
diff between pab___________________________:30 and pac___________________________:30 = -12 ; known: pa
pab___________________________ and pac___________________________ have different instruction count: -12
diff between pac!__________________________:30 and pac"__________________________:30 = 0 ; known: pac
diff between pac"__________________________:30 and pac#__________________________:30 = 0 ; known: pac
.
.
.
diff between packc_________________________:30 and packd_________________________:30 = 0 ; known: pack
diff between packd_________________________:30 and packe_________________________:30 = -12 ; known: pack
packd_________________________ and packe_________________________ have different instruction count: -12
diff between packe!________________________:30 and packe"________________________:30 = 0 ; known: packe
diff between packe"________________________:30 and packe#________________________:30 = 0 ; known: packe
.
.
.
diff between packers_and_vms_and_xors_oh_mv:30 and packers_and_vms_and_xors_oh_mw:30 = 0 ; known: packers_and_vms_and_xors_oh_m
diff between packers_and_vms_and_xors_oh_mw:30 and packers_and_vms_and_xors_oh_mx:30 = 0 ; known: packers_and_vms_and_xors_oh_m
diff between packers_and_vms_and_xors_oh_mx:30 and packers_and_vms_and_xors_oh_my:30 = -2839 ; known: packers_and_vms_and_xors_oh_m
packers_and_vms_and_xors_oh_mx and packers_and_vms_and_xors_oh_my have different instruction count: -2839
diff between packers_and_vms_and_xors_oh_my!:31 and packers_and_vms_and_xors_oh_my":31 = 0 ; known: packers_and_vms_and_xors_oh_my
diff between packers_and_vms_and_xors_oh_my":31 and packers_and_vms_and_xors_oh_my#:31 = 0 ; known: packers_and_vms_and_xors_oh_my

```

As it finds each character, it prints it out under "known". After some time, we had "packers_and_vms_and_xors_o" and I just guessed the rest as "oh_my", which turned out to be correct. The flag is **packers_and_vms_and_xors_oh_my**


#### The not-so smart way...

barrebas here, enlightening you on the idiot way of solving baleful. The pseudocode generated by Hopper and other tools was nigh unreadable. We quickly identified that we were up against some kind of virtual machine and that bytecode instructions are being processed by the assembly. The instructions were decoded and then control was handed of the appropriate assembly code here:

```
   0x8048a44:	mov    eax,DWORD PTR [eax*4+0x8049dd4]
   0x8048a4b:	jmp    eax
   0x8048a4d:	add    DWORD PTR [ebp-0x34],0x1
```

The `jmp eax` then jumps to the correct block of assembly to handle that virtual machine instruction. 

superkojiman told me that the password was 30 characters, and so I tried to identify the chunks of code that were operating on the password. First, I thought I'd step throught the code with `gdb`, but as superkojiman already showed, more than 40,000 instructions were executed. Not gonna do that manually! Instead, I used some python magic with gdb to dump the contents of `eax` at `0x8048a4b`:

```python
class DebugPrintingBreakpoint(gdb.Breakpoint):
    def stop(self):
		with open('calls', 'a') as f:
			f.write("call: {}\n".format(gdb.parse_and_eval("$eax")))
			f.close()
		return False # do not drop to gdb's prompt
		
with open('calls', 'w') as f:
	f.write('starting to trace baleful\n')
	f.close()
	
gdb.execute("start")
DebugPrintingBreakpoint("*0x8048a4b")
gdb.execute("continue")
```

This could be ran from within `gdb` like so:

```bash
gdb-peda$ source baleful-dump.py
```

The execution of the program will stop to ask for a password. I ran this script twice: once for a 0-length password and once for a 30-length password. This generated two files and with `sort | uniq` I was able to identify three addresses that were specific to the password-checking code:

```
0x8048a4d
0x8048e2f
0x80497b9
```

Backtracking throught the code, it seems that our chars are being mangled for comparison here;

```
   0x80499f6:	sub    ecx,eax
   0x80499f8:	mov    eax,ecx
   0x80499fa:	mov    DWORD PTR [ebp-0x28],eax
```

This value at 0x28 is later checked for NULL. I modified the python script to dump the contents of `ecx` at this postion:

```python
class DebugPrintingBreakpoint(gdb.Breakpoint):
    def stop(self):
		with open('ecx-values', 'a') as f:
			f.write("eax: {}\n".format(gdb.parse_and_eval("$eax")))
			f.write("ecx: {}\n".format(gdb.parse_and_eval("$ecx")))
			f.close()
		return False # do not drop to gdb's prompt

with open('ecx-values', 'w') as f:
	f.write('starting to trace baleful\n')
	f.close()
	
gdb.execute("start")
DebugPrintingBreakpoint("*0x080499f6")
gdb.execute("continue")
```

Now, this output contained two thousand of lines. Oops! It turns out that this virtual machine instruction is a generic "subtract a from b". Nevertheless, using the same trick as before (0 length password vs 30 length password) and by changing the supplied password a bit, I was able to find where the code started to compare the characters:

```
eax: 0x8d	# first char
ecx: 0x8d
eax: 0x1e	# check the length of the password
ecx: 0x1
eax: 0x6f	# second char
ecx: 0x6f	
eax: 0x1e	# check the length of the password
ecx: 0x2
eax: 0x0	# third char
ecx: 0x0
```

Through some *ahem* guesswork, it looked like the first char is `p`. The second one was `a`. I guessed `c` and `k`. I continued, painstakingly, each time manually adjusting the guessed password and checking the output of the script. Turns out the flag looked like `packers_and_vms_and_xor`. Take would explain why I couldn't find a one-to-one relation between the characters I entered and the ones that ended up in the dumped file. With that in mind, I ran this piece of python to extract the xor key:

```python
>>> print [chr(ord(x) ^ ord(y)) for x,y in zip("packers", "\x8d\x6f\x00\x24\x98\x7c")]
['\xfd', '\x0e', 'c', 'O', '\xfd', '\x0e']
```

So the values are xor’ed with these bytes. I grabbed the rest of the values:

```python
>>> print [chr(ord(x) ^ ord(y)) for x,y in zip("\xfd\x0ecO"*10, "\x8d\x6f\x00\x24\x98\x7c\x10\x10\x9c\x60\x07\x10\x8b\x63\x10\x10\x9c\x60\x07\x10\x85\x61\x11\x3c\xa2\x61\x0b\x10\x90\x77")]
['p', 'a', 'c', 'k', 'e', 'r', 's', '_', 'a', 'n', 'd', '_', 'v', 'm', 's', '_', 'a', 'n', 'd', '_', 'x', 'o', 'r', 's', '_', 'o', 'h', '_', 'm', 'y']
```

Which turned out to be the flag, but I got it just seconds after superkojiman figured it out with the pintool. D'oh! Black-box reverse-engineering is painful!

