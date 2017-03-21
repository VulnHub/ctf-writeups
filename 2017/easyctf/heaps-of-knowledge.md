### Solved by superkojiman

This was a 420 point binary exploitation challenge. The challenge's name, Heaps of Knowledge, hints that it's probably a heap based vulnerability. So let's have a quick look. The binary presents us with a menu where we can edit, delete, and print chapters in a book. Here's what a sample run looks like:

![](/images/2017/easyctf/heaps-of-knowledge/01.png)

And here's a run that triggers a segmentation fault:

![](/images/2017/easyctf/heaps-of-knowledge/02.png)

From the looks of it, editing the first chunk somehow overwrote something important in the second chunk and caused the segmentation fault. A quick look at the core dump shows that EIP was overwritten with "cccc":

![](/images/2017/easyctf/heaps-of-knowledge/03.png)

We have control over execution, half the challenge has been sovled! So what's actually happening? 

When we edit a chapter, a book structure is populated with the chapter number, title, and a function pointer to display the text in that chapter. The function pointer points to the `print_chapter()` function, which is called when the publish option in the menu is selected. Here's what it looks like on the heap after creating two chapters:

![](/images/2017/easyctf/heaps-of-knowledge/07.png)

The data highlighed in red show the function pointer and text belonging to chapter 1. The data highlighted in blue show the function pointer and text belonging to chapter 2. Take note of chapter 2's function pointer located at 0x98bc880 as we will overwrite this.

If we choose to edit chapter 1 again, it won't create a new chunk; rather it will just ask us to enter new text:

![](/images/2017/easyctf/heaps-of-knowledge/08.png)

Notice that it uses `gets()` to read input. `gets()` doesn't do bounds checking so we can write well past this chunk and into the next chunk, thereby overwriting the function pointer assigned to chapter 2. If we set a breakpoint at the call to `gets()` we see that it is going to copy the next text into the buffer containing the old text:

![](/images/2017/easyctf/heaps-of-knowledge/09.png)

This happens to be 32 bytes from the location of chapter 2's function pointer. So if we overwrite this pointer with the address of a function, then when we print the contens of the book, the function of our choosing will be executed. So what function should we call?

A quick examination of the binary shows that a function called `give_flag()` exists and isn't called anywhere. The function as you may have guessed, will open a `flag.txt` on the server and print out the flag. 

![](/images/2017/easyctf/heaps-of-knowledge/04.png)

Let's test it. I entered 32 "c"s followed by 4 "d"s. This should overwrite chapter 2's function pointer with 0x64646464:

![](/images/2017/easyctf/heaps-of-knowledge/10.png)

Notice that the function pointer at 0x98bc880 has been overwritten. Now if we select the publish option, it will print call the function pointers in all the chapters, including the one we've overwritten.

So all we need to do is replace the chapter 1's text  with the address of `give_flag()` and we should get the flag. I created a `flag.txt` with a dummy flag inside and ran my exploit:

![](/images/2017/easyctf/heaps-of-knowledge/05.png)

Oh look it worked! Running it against the EasyCTF server nets us the flag:

![](/images/2017/easyctf/heaps-of-knowledge/06.png)

