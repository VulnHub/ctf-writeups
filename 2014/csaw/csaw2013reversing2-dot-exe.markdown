### Solved by superkojiman

Time to switch to a Windows VM for this one. A binary csaw2013reversing2.exe is provided along with the following hint:

> We got a little lazy so we just tweaked an old one a bit

I executed the binary and was greeted with the following message box dialog:

![](/images/2014/csaw/csaw2013reversing2/01.png)

So it looks like the flag is being printed out, except it's encrypted. I loaded it up in IDA Pro and examined the _main function. One of the first things that stood out was a call to isDebuggerPresent():

![](/images/2014/csaw/csaw2013reversing2/02.png)

This function checks to see if the binary is being run within a debugger, and if it is, it takes the branch on the right which leads to a interrupt instruction (`int 3`) that triggers an exception handler that terminates the program. What's interesting is right after that instruction, there's call made to a function `sub_401000()` that never gets executed.

Examining `sub_401000()` shows the following:

![](/images/2014/csaw/csaw2013reversing2/03.png)

So what does this function do? It looked like it was XORing the contents of `[eax+ecx*4]` with the value in `esi` in a loop, which made me suspect that it was the function responsible for decrypting the flag.  I went ahead and ran the program in IDA's debugger and as expected, I hit the software breakpoint which terminated the program.


![](/images/2014/csaw/csaw2013reversing2/04.png)

To bypass this, I set a breakpoint at 0x00401099 which is the `inc ecx` instruction right before `int 3`, and ran the program again. This time round, I hit the breakpoint right before `int 3` as expected, and changed the value of `eip` to 0x0040109b, which is the instruction right after `int 3`

![](/images/2014/csaw/csaw2013reversing2/05.png)

I could now step into the `sub_401000` function to see what was going on.

Here's what it looks like right after stepping into `sub_401000`.

![](/images/2014/csaw/csaw2013reversing2/06.png)

I was interested in what the memory location referred to at `[edx+ecx*4]` was. This was the memory location that was being XORd in a loop with the value in `esi` (see address 0x0040101f). Hovering over this address showed that it pointed to 0x00381eac. I pulled up that location in the dump window:

![](/images/2014/csaw/csaw2013reversing2/07.png)

That looks like it might be the encrypted flag! I set a breakpoint at 0x0040101f and continued execution. From here on I stepped over each instruction in the loop and watched the bytes in 0x00381eac transform into the flag:

![](/images/2014/csaw/csaw2013reversing2/08.png)

And so the flag is: **flag{reversing_is_not_that_hard}**

