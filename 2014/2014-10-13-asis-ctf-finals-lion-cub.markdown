### Solved by superkojiman and barrebas

Lion Cub is a 200 point reversing challenge for the ASIS CTF Finals 2014. A file called simple_f0455e55c1d236a28387d04d5a8672ad was provided for download, along with the following description:

> Flag is encrypted using this program, can you find it?

This file is an archive that contains a directory called simple. However the file was archived multiple times using xz, gzip,and tar. So let's go ahead and unpack it.

First, unxz:

```
root@kali ~/asis-ctf/lioncub
# file simple_f0455e55c1d236a28387d04d5a8672ad
simple_f0455e55c1d236a28387d04d5a8672ad: XZ compressed data

root@kali ~/asis-ctf/lioncub
# mv simple_f0455e55c1d236a28387d04d5a8672ad simple_f0455e55c1d236a28387d04d5a8672ad.xz

root@kali ~/asis-ctf/lioncub
# unxz simple_f0455e55c1d236a28387d04d5a8672ad.xz
```

Then gunzip:

```
root@kali ~/asis-ctf/lioncub
# file simple_f0455e55c1d236a28387d04d5a8672ad
simple_f0455e55c1d236a28387d04d5a8672ad: gzip compressed data, from Unix, last modified: Sat Oct 11 05:44:23 2014

root@kali ~/asis-ctf/lioncub
# mv simple_f0455e55c1d236a28387d04d5a8672ad simple_f0455e55c1d236a28387d04d5a8672ad.gz

root@kali ~/asis-ctf/lioncub
# gunzip simple_f0455e55c1d236a28387d04d5a8672ad.gz
```

Then tar:

```
root@kali ~/asis-ctf/lioncub
# tar xvf simple_f0455e55c1d236a28387d04d5a8672ad
simple/
simple/simple_5c4d29f0e7eeefd7c770a22a93a1daa9
simple/flag.enc
```

The end result is a directory called simple which contains two files. flag.enc is the encrypted flag. The other file is another xz compressed simple_5c4d29f0e7eeefd7c770a22a93a1daa9, which is the 64-bit binary used to generate flag.enc:

```
root@kali ~/asis-ctf/lioncub/simple
# mv simple_5c4d29f0e7eeefd7c770a22a93a1daa9 simple_5c4d29f0e7eeefd7c770a22a93a1daa9.xz

root@kali ~/asis-ctf/lioncub/simple
# unxz simple_5c4d29f0e7eeefd7c770a22a93a1daa9.xz

root@kali ~/asis-ctf/lioncub/simple
# file simple_5c4d29f0e7eeefd7c770a22a93a1daa9
simple_5c4d29f0e7eeefd7c770a22a93a1daa9: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.26, BuildID[sha1]=0x65637aabfb02062b1ab3735ef1cbcea019b76b46, stripped
```

I started off by checking for any interesting strings in the binary:

```
root@kali ~/asis-ctf/lioncub/simple
# strings simple_5c4d29f0e7eeefd7c770a22a93a1daa9
.
.
.
flag
flag.enc
.
.
.
```

Only two strings of interest were found; flag and flag.enc. I went ahead and ran the binary, and it instantly aborted:

```
root@kali ~/asis-ctf/lioncub/simple
# ./simple_5c4d29f0e7eeefd7c770a22a93a1daa9
terminate called after throwing an instance of 'std::bad_alloc'
  what():  std::bad_alloc
  Aborted
```

Time to see what was going on. I loaded the binary in gdb, punched in start, and execution paused at the entry point at 0x400bfc:

```
   0x400bf5:    jmp    0x400b70
   0x400bfa:    nop
   0x400bfb:    nop
=> 0x400bfc:    push   rbp
   0x400bfd:    mov    rbp,rsp
   0x400c00:    push   rbx
   0x400c01:    sub    rsp,0x448
   0x400c08:    mov    esi,0x4
```

I opened the binary in Hopper and located the address 0x400bfc and decompiled it into the following:

```c
int sub_400bfc() {
    LODWORD(rdx) = LODWORD(sub_400de1(LODWORD(sub_400de1(0x8, 0x4)), 0x2));
    std::basic_ifstream<char, std::char_traits<char> >::basic_ifstream();
    std::basic_ofstream<char, std::char_traits<char> >::basic_ofstream();
    sub_400df6(var_30);
    var_450 = std::istream::tellg();
    var_448 = rdx;
    var_30 = var_450;
    var_28 = var_448;
    sub_400e22(var_30);
    var_20 = operator new[]();
    std::istream::seekg();
    sub_400e22(var_30);
    std::istream::read();
    std::basic_ofstream<char, std::char_traits<char> >::open();
    var_14 = 0x0;
    do {
            rax = sub_400e22(var_30);
            if (LOBYTE(LODWORD(LODWORD(rax) - 0x1) <= var_14 ? 0x1 : 0x0) == 0x0) {
                break;
            }
            std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >();
            var_14 = var_14 + 0x1;
    } while (true);
    std::basic_ofstream<char, std::char_traits<char> >::close();
    if (var_20 != 0x0) {
            operator delete[]();
    }
    std::basic_ofstream<char, std::char_traits<char> >::~basic_ofstream();
    std::basic_ifstream<char, std::char_traits<char> >::~basic_ifstream();
    LODWORD(rax) = LODWORD(0x0);
    return rax;
}
```

Looks like it's dealing with file input and output, which is to be expected. More interestingly, a loop in the middle of the function that could be doing the encryption. I looked for the strings "flag" and "flag.enc" as well, and noticed that "flag" was being referenced in 0x400bfc+48. Here it is at 0x4002c2:

![](/images/2014/asis-finals/lioncub/01.png)

I set a breakpoint at 0x400c2c and ran the binary again.

```
   0x400c1e:    call   0x400de1
   0x400c23:    mov    edx,eax
   0x400c25:    lea    rax,[rbp-0x240]
=> 0x400c2c:    mov    esi,0x400eec
   0x400c31:    mov    rdi,rax
   0x400c34:
    call   0x400ab0 <_ZNSt14basic_ifstreamIcSt11char_traitsIcEEC1EPKcSt13_Ios_Openmode@plt>
   0x400c39:    lea    rax,[rbp-0x440]
   0x400c40:    mov    rdi,rax
```
I stepped through execution and stopped at 0x400c34. gdb assumes the following arguments are being passed to _ZNSt14basic_ifstreamIcSt11char_traitsIcEEC1EPKcSt13_Ios_Openmode

```
Guessed arguments:
arg[0]: 0x7fff06ca6d50 --> 0x0
arg[1]: 0x400eec --> 0x616c660067616c66 ('flag')
arg[2]: 0xe
```

So it's looking for a file called "flag", which doesn't exist. That could be why the program is crashing. To test it out, I created a new file called "flag" which contained "ABCD" and ran the program again:

```
root@kali ~/asis-ctf/lioncub/simple
# echo "ABCD" > flag

root@kali ~/asis-ctf/lioncub/simple
# ./simple_5c4d29f0e7eeefd7c770a22a93a1daa9

root@kali ~/asis-ctf/lioncub/simple
#
```

Hey, no crash, but it looks like flag.enc has been modified. It's significantly smaller than before, which makes sense. The binary takes a file called flag, encrypts its contents, and saves it as flag.enc. Checking out the contents of flag.enc showed:

```
root@kali ~/asis-ctf/lioncub/simple
# xxd -g1 flag.enc
0000000: 03 01 07 4e                                      ...N
```

flag.enc contains four bytes, much like the input file flag. It looked like each byte was being encrypted, so I assumed it was some simple substition cipher or XOR encryption.

After stepping through execution in gdb and examining the code, I identified the location where the contents of the input file were being encrypted. The code block is at 0x400d05

```
gdb-peda$ x/20i 0x400d05
=> 0x400d05:    mov    eax,DWORD PTR [rbp-0x14]
   0x400d08:    movsxd rdx,eax
   0x400d0b:    mov    rax,QWORD PTR [rbp-0x20]
   0x400d0f:    add    rax,rdx
   0x400d12:    movzx  edx,BYTE PTR [rax]
   0x400d15:    mov    eax,DWORD PTR [rbp-0x14]
   0x400d18:    cdqe
   0x400d1a:    lea    rcx,[rax+0x1]
   0x400d1e:    mov    rax,QWORD PTR [rbp-0x20]
   0x400d22:    add    rax,rcx
   0x400d25:    movzx  eax,BYTE PTR [rax]
   0x400d28:    xor    eax,edx
   0x400d2a:    movsx  edx,al
   0x400d2d:    lea    rax,[rbp-0x440]
   0x400d34:    mov    esi,edx
   0x400d36:    mov    rdi,rax
   0x400d39:    call   0x400a40 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_c@plt>
   0x400d3e:    add    DWORD PTR [rbp-0x14],0x1
```

In Hopper, this code block corresponds to the following graph:

![](/images/2014/asis-finals/lioncub/02.png)

Here's what's happening. At 0x400d0b, the a pointer to the contents of the input file is copied into RAX.

```
=> 0x400d0b:    mov    rax,QWORD PTR [rbp-0x20]
```

RAX now contains a pointer to "ABCD". Next the first byte in the input file is copied into EDX:

```
=> 0x400d12:    movzx  edx,BYTE PTR [rax]
```

RAX now contanins the value 'A'. The contents of the input file are loaded into RAX once again, and the value in RCX is added to RAX. RCX contains 1, so it basically just increments the pointer in RAX so that it contains "BCD" instead of "ABCD". The first byte of that string is copied into RAX.

```
=> 0x400d1e:    mov    rax,QWORD PTR [rbp-0x20]
   0x400d22:    add    rax,rcx
```

RAX now contains 'B'. Finally, RAX and RDX are XOR'd:

```
=> 0x400d28:    xor    eax,edx
```

The result is stored in RAX, which is 0x3. I took a look at the contents of flag.enc again:

```
# xxd -g1 flag.enc
0000000: 03 01 07 4e                                      ...N
```

First byte is 0x3! So the encryption is relatively simple. Each byte is XOR'd with its next byte.

```
0x41 ^ 0x42  = 0x03
0x43 ^ 0x44  = 0x01
0x44 ^ 0x45  = 0x07
0x45 ^ 0x0a  = 0x4e
```

I knew how the input was being encrypted, but how to decrypt flag.enc? For that, I handed it off to barrebas.

Barrebas here. Superkojiman did one hell of a job again and I just took the last hurdle. Initially, I thought we could decrypt the flag by breaking the crypto, but I was wrong. Without knowing any byte (preferably the first), the encoding scheme is not breakable. Furthermore, ```flag.enc``` looks huge. It's unlikely that it's a text file. After a small break I figured it was more likely that it was something like an image, like a BMP. Without knowing the first byte a priori, I resorted to bruteforcing the first byte. Using that guessed first byte, I took the second _encoded_ byte of ```flag.enc```, XOR'd it and got the second decoded byte. That second decoded byte XOR'd with the third _encoded_ byte from flag.enc gives the third decoded byte, and so on. I wrote the following Python script:

```
#!/usr/bin/python

with open('flag.enc') as f:
	o_data = f.read()
	f.close()

for k in range(256):
	data = o_data
	decoded = chr(k)

	nextChar = k
	for i in range(len(data)):
		nextChar = ord(data[i]) ^ nextChar
		decoded += chr(nextChar)

	with open(chr(k)), 'w') as out:
		out.write(decoded)
```

This loops through all the possible first bytes, decodes the data in the manner superkojiman described and dumps the data in a file. Next, I ran file on all the decoded files:

![](/images/2014/asis-finals/lioncub/03.png)

That ```31``` file looked interesting, seeing as it was a gzipped file containing... flag.png? Let's extract it!

![](/images/2014/asis-finals/lioncub/04.png)

Ah, a QR code. Surely, the flag was in there! I scanned it with my phone and got:

```text
1f8b0808928d2f540003666c61672e706e67000192016dfe89504e470d0a
1a0a0000000d494844520000006f0000006f0103000000d80b0c23000000
06504c5445000000ffffffa5d99fdd0000000274524e53ffffc8b5dfc700
0000097048597300000b1200000b1201d2dd7efc0000012449444154388d
d5d431ae84201006e079b1a0d30b90cc35e8b8925e40e5027a253aae41e2
05b0a320ce1bb2bbef651b87668b25167e0501867f007a1bf01d4c004be8
76af0150e449650a71087aa1067afee9762aa36ae2a8746f7523674bbb2f
4de4250c12fd6ff2867cde2968fefe8e7f431e17943af155d81b26d06068
b3dd661a68d005987cfc219997e23b8ab3c24bc9a4808e3acab4da065204
a541e166506402e4592ec71148e499cbe0f914b42a99c91eb59221824591
e4613475b9dd93c804e4087a13472bf3ac05e7781f6b0b0326551be77147
02b35e38540a0064f2481c6fc2d3cbe4c472bc5dd613c9e45e98a10c19cf
576bdc915b9213cbbb524d9c88d73ab667ad44d667e419957b72ffdace79
bc8ccc47fff696eb2ff3734feea7f80bb686232e7a493424000000004945
4e44ae426082fb73fb8e92010000
```

Hmm. I dumped this into a file using ```xxd```:

```
echo -ne '1f8b0808928d2f540003666c61672e706e67000192016dfe89
504e470d0a1a0a0000000d494844520000006f0000006f0103000000d80b
0c2300000006504c5445000000ffffffa5d99fdd0000000274524e53ffff
c8b5dfc7000000097048597300000b1200000b1201d2dd7efc0000012449
444154388dd5d431ae84201006e079b1a0d30b90cc35e8b8925e40e5027a
253aae41e205b0a320ce1bb2bbef651b87668b25167e0501867f007a1bf0
1d4c004be876af0150e449650a71087aa1067afee9762aa36ae2a8746f75
23674bbb2f4de4250c12fd6ff2867cde2968fefe8e7f431e17943af155d8
1b26d06068b3dd661a68d005987cfc219997e23b8ab3c24bc9a4808e3aca
b4da065204a541e166506402e4592ec71148e499cbe0f914b42a99c91eb5
9221824591e4613475b9dd93c804e4087a13472bf3ac05e7781f6b0b0326
551be7714702b35e38540a0064f2481c6fc2d3cbe4c472bc5dd613c9e45e
98a10c19cf576bdc915b9213cbbb524d9c88d73ab667ad44d667e419957b
72ffdace79bc8ccc47fff696eb2ff3734feea7f80bb686232e7a49342400
00000049454e44ae426082fb73fb8e92010000' |xxd -r -p > flag2
```

```file``` revealed yet another gzipped file. After decompression, I found another PNG:

![](/images/2014/asis-finals/lioncub/05.png)

I scanned it again, and this time, it was jackpot!

Flag: ```ASIS_e87b556efc59f8351aec0858da850906```

Again, I couldn't have gotten the flag without the brilliant reverse-engineering-fu of superkojiman!
