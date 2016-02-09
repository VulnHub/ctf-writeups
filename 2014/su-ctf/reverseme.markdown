### Solved by superkojiman and barrebas

This is a 200 point reversing challenge for Sharif CTF 2014 called ReverseMe.exe. The hint for this challenge is:

> Reverse me and find a valid serial number! flag : [A valid serial number]

So the goal is to find the serial number.

A dialog box is presented to the user when the binary is executed:

![](/images/2014/sharif/reverseme/01.png)

Quick testing showed that if an invalid or blank email address was entered, the binary would complain with:

![](/images/2014/sharif/reverseme/02.png)

Entering a valid email address and an incorrect serial number would pop up the following:

![](/images/2014/sharif/reverseme/03.png)

It seemed like one of those classic crackme binaries. I opened it up in IDA to have a better look at what I was up against. As it was loading the binary, the following message popped up:

![](/images/2014/sharif/reverseme/04.png)

Ok, not good. It looked like the binary had been packed. I loaded it up in PEiD and it confirmed that the packer used was ASPack 2.12.

![](/images/2014/sharif/reverseme/05.png)

The first step was to unpack it the binary. ASPack 2.12's packing algorithm is pretty well documented online, so I went ahead and did this part manually. First, I loaded the binary in OllyDbg.

![](/images/2014/sharif/reverseme/06.png)

The first instruction was a pushad. I executed this, and then in the hex dump window, I Ctrl-G and entered esp in the popup window:

![](/images/2014/sharif/reverseme/07.png)

This brought me to the following memory location:

![](/images/2014/sharif/reverseme/08.png)

I highlighted the first four bytes (28 02 91 7C), right clicked on it, and selected Breakpoint > Hardware, on access > DWORD.

![](/images/2014/sharif/reverseme/09.png)

I let the program continue running until it hit the breakpoint. At this point I was now at the following instruction:

![](/images/2014/sharif/reverseme/10.png)

This jump instruction would be taken to 0x00417420 where it would push the address of OEP onto the stack and return to it. So I stepped through the instructions three times until it returned and I was left with the following:

![](/images/2014/sharif/reverseme/11.png)

The binary was unpacked at this point, but it had to be reanalysed. Hitting Ctrl-A in the disassembler window fixed this:

![](/images/2014/sharif/reverseme/12.png)

Much better! I launched the OllyDump plugin, unchecked the Rebuild Import box, clicked Dump, and saved it as Dump-ReverseMe.exe. Note that it detected OEP as 16BB.

![](/images/2014/sharif/reverseme/13.png)

Now I had to rebuild the IAT. I launched Import REConstructor and attached to the ReverseMe.exe process that OllyDbg executed.

![](/images/2014/sharif/reverseme/14.png)

I set OEP to 16BB and clicked IAT AutoSearch, and then clicked on Get Imports. The Imported Functions Found window now displayed the entries for kernel32.dll and user32.dll:

![](/images/2014/sharif/reverseme/15.png)

I clicked on Fix Dump, selected Dump-ReverseMe.exe, and it saved the unpacked binary as Dump-ReverseMe_.exe.

At this point the binary should be properly unpacked, so I loaded it into IDA once again. No warning messages popped up and I could now see the full list of functions the binary was using.

This is the packed binary loaded into IDA:

![](/images/2014/sharif/reverseme/16.png)

And here it is unpacked:

![](/images/2014/sharif/reverseme/17.png)

After all that, I could now get started on cracking the serial number. I looked for any interesting strings and found a couple:

![](/images/2014/sharif/reverseme/18.png)

I followed the "Registeration" string and examined the xrefs to this memory location. It was being use db the function DialogFunc():

![](/images/2014/sharif/reverseme/19.png)

So looked into the DialogFunc() function and found the code that checked to see if the email address was valid, and if it was, the branch that checked the serial number:

![](/images/2014/sharif/reverseme/20.png)

The serial number check was relatively simple. It examined each character, did an optional operation on it (like adding a value to it), and checking the result against a value it expected. If it got what it was expecting, then it moved on to the next character in the  serial number and did a similar check. Here's a sample of the checks it does:

![](/images/2014/sharif/reverseme/21.png)

I went through the checks, manually setting ZF to the correct value in order to continue down the chain of checks until I ended up with the key **B`9dmq4c8g9G7b;Y** which I submitted and was told was wrong.

As it turned out, I was quite close, but I must have calculated a couple of those characters incorrectly. By this time I decided to hand over my findings to barrebas as I had other commitments to attend to and knew I wouldn't have time to work on the binary any longer. I'll let him explain from here on.

OK, barrebas here. I fired up a Windows XP VM so I could debug this beautifully unpacked binary. After loading the dumped exe in Ollydebug (what a great piece of software!) I pressed F9 to run it and was greeted with the Registration dialog. I entered a bogus email and serial number and got a MessageBox:

![](/images/2014/sharif/reverseme/22.png)

That's fine and dandy. I had somewhere to hook into the code execution. In Olly, I set a breakpoint on MessageBoxA and entered another bogus email address and serial. The binary hit the breakpoint and I could backtrace:

![](/images/2014/sharif/reverseme/23.png)

I followed the JMPs back and landed at a critical point, where at lot of JMPs pointed. Furthermore, the block of code above this point seems to use the MOVSX instruction to move a single byte into EAX. It then compares it to a value that looks suspiciously like an ASCII value! All those instructions, comparing bytes... comparing bytes of our serial maybe? I named this piece of code "FAIL", to make the jumps easier to read.

![](/images/2014/sharif/reverseme/24.png)

I went to the first JMP and set a breakpoint at 0x40119B: CMP ECX, 10. The block of code just above that appears to get a string length and guess for what:

![](/images/2014/sharif/reverseme/25.png)

That's our serial number! So the serial is supposed to be 16 bytes long. I entered "0123456789abcdef" as a serial. The CMP ECX, 10/JNZ will not fire and we get into the first comparision block.

![](/images/2014/sharif/reverseme/26.png)

So this first check is for position '0', the first byte of our serial number. The code compares it to the hex value 0x42, or 'B'. The first character of the serial must therefore be 'B'. I hexedited the entered serial so that the check will pass. Next, the code takes the value 'f', the last position of the serial, adds 0x42 to it (the first character) and compares it to 0x9B. Reversing this operation, 0x9B-0x42 = 0x59, or 'Y'. Once again, hexediting the serial so the check passes. The code then takes the value '1' from our serial, the second byte. The LEA instruction, in this case, does nothing more that subtracting 3. The value is then compared to 0x57, so the proper byte should be 0x5A or 'Z'. Indeed, this works! The code continually does this, zig-zagging across the serial: first byte, last byte, second byte, first to last byte, third byte, second to last byte... Comparing these values to hard-coded values. It's easy to reverse!

![](/images/2014/sharif/reverseme/27.png)

The serial check function is actually really simple. The email address is not used to generate the serial, the code only checks the serial versus hard-coded values. Continuing this process down the code leads to the serial: BZ9dmq4c8g9G7bAY. This turns out to be the flag as well!

![](/images/2014/sharif/reverseme/28.png)

Quite simple really, the hard part was definitely done by superkojiman who manually unpacked the entire binary!


