### Solved by barrebas

The last flag for 9447 CTF that I got was this binary reversing challenge. Let's get `rolling`!

Identifying the binary with file showed that it was a 64-bit ELF, dynamically linked. Unfortunately for me, it was linked against a higher `libc` version:

```bash
bas@tritonal:~/tmp/9447$ ./rolling 
./rolling: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.14' not found (required by ./rolling)
```
To solve this issue, I needed a way to get the program to use a newer version of libc. One way to do this is using `LD_PRELOAD`. I downloaded a [newer libc deb](http://pkgs.org/ubuntu-14.10/ubuntu-main-amd64/libc6_2.19-10ubuntu2_amd64.deb.html), that ought to be binary compatible with my debian box. After unpacking `ld-2.19.so` and `libc-2.19.so`, I could start the binary like this:

```bash
bas@tritonal:~/tmp/9447$ LD_PRELOAD=./libc-2.19.so ./ld-2.19.so ./rolling
Fynd i mewn i cyfrinair
```

And in `gdb`:

```bash
gdb-peda$ set environment LD_PRELOAD=./libc-2.19.so ./ld-2.19.so
gdb-peda$ r
Fynd i mewn i cyfrinair

Program received signal SIGSEGV, Segmentation fault.
<snip>
```

The program would still segfault, but at least it ran. Okay, let's get to work. The strange string meant nothing to me, but it's Welsh for "Enter a password". Of course, the description on 9447 mentioned that the binary would take an input. The flag is the input which the binary accepts. I ran the binary with an argument, which resulted in another Welsh string. `strings` identified the last Welsh string. I looked up their meaning via Google Translate and their address in `gdb`:

```bash
Nac oes. Ceisiwch eto. == No. Try again. // rolling : 0x600865 ("Nac oes. Ceisiwch eto.")
Llongyfarchiadau == Congratulations // rolling : 0x600854 ("Llongyfarchiadau")
```

These strings look like the "Good"/"Bad" output that we expect for this input-checking binary! Switching over to the output of `objdump`, I looked up where these strings are referenced:

```bash
  400771:	48 8b 55 f0          	mov    -0x10(%rbp),%rdx
  400775:	48 83 c2 08          	add    $0x8,%rdx
  400779:	48 8b 12             	mov    (%rdx),%rdx
  40077c:	48 89 d7             	mov    %rdx,%rdi
  40077f:	ff d0                	callq  *%rax		# interesting function
  400781:	85 c0                	test   %eax,%eax	# if eax == 1 -> success
  400783:	74 0c                	je     400791 <memcpy@plt+0x2b1>
  400785:	bf 54 08 40 00       	mov    $0x400854,%edi				# Llong...
  40078a:	e8 11 fd ff ff       	callq  4004a0 <puts@plt>
  40078f:	eb 16                	jmp    4007a7 <memcpy@plt+0x2c7>
  400791:	bf 65 08 40 00       	mov    $0x400865,%edi				# Nac oes... 
  400796:	e8 05 fd ff ff       	callq  4004a0 <puts@plt>
  40079b:	eb 0a                	jmp    4007a7 <memcpy@plt+0x2c7>
  40079d:	bf 7c 08 40 00       	mov    $0x40087c,%edi
  4007a2:	e8 f9 fc ff ff       	callq  4004a0 <puts@plt>
  4007a7:	b8 00 00 00 00       	mov    $0x0,%eax
  4007ac:	c9                   	leaveq 
  4007ad:	c3                   	retq  
```

The `test eax, eax` at `0x400781` controls which path is taken: either OK ("Llong...") or not OK ("Nac oes..."). The value of `eax` is probably set by the function that is called at `0x40077f: callq  *%rax`. Switching back to `gdb`, I set a breakpoint on `0x40077f` and prepared to trace that function. 

```bash
gdb-peda$ b *0x40077f
Breakpoint 1 at 0x40077f
gdb-peda$ r bleh
...
[-------------------------------------code-------------------------------------]
   0x400775:	add    rdx,0x8
   0x400779:	mov    rdx,QWORD PTR [rdx]
   0x40077c:	mov    rdi,rdx
=> 0x40077f:	call   rax
   0x400781:	test   eax,eax
   0x400783:	je     0x400791
...
Breakpoint 1, 0x000000000040077f in ?? ()
```

The binary was halted at the `call eax` instruction. I entered `ni` to step into the function. This is where the fun really starts, it is where our string is checked for validity. There's a red herring in there too. The function starts like this:

```
gdb-peda$ x/40i $rip
=> 0x7ffff7ff5000:	push   rbp
   0x7ffff7ff5001:	mov    rbp,rsp
   0x7ffff7ff5004:	sub    rsp,0x10
   0x7ffff7ff5008:	mov    QWORD PTR [rbp-0x8],rdi
   0x7ffff7ff500c:	mov    rax,QWORD PTR [rbp-0x8]
   # grab first byte of input
   0x7ffff7ff5010:	movzx  eax,BYTE PTR [rax]	
   # is it '9'?
   0x7ffff7ff5013:	cmp    al,0x39
   # if so, jump away
   0x7ffff7ff5015:	je     0x7ffff7ff5143	
   # else:
   0x7ffff7ff501b:	mov    rax,QWORD PTR [rbp-0x8] 
   # grab first byte of input
   0x7ffff7ff501f:	movzx  eax,BYTE PTR [rax]	
   # is it 'f'?
   0x7ffff7ff5022:	cmp    al,0x66			
   # if not, jump away
   0x7ffff7ff5024:	jne    0x7ffff7ff5139	
   0x7ffff7ff502a:	mov    rax,QWORD PTR [rbp-0x8]
   # second byte of input
   0x7ffff7ff502e:	add    rax,0x1		
   0x7ffff7ff5032:	movzx  eax,BYTE PTR [rax]
   # is it 'l'?
   0x7ffff7ff5035:	cmp    al,0x6c	
   0x7ffff7ff5037:	jne    0x7ffff7ff5139
   0x7ffff7ff503d:	mov    rax,QWORD PTR [rbp-0x8]
   # third byte of input
   0x7ffff7ff5041:	add    rax,0x2
   0x7ffff7ff5045:	movzx  eax,BYTE PTR [rax]
   # is it 'a'?
   0x7ffff7ff5048:	cmp    al,0x61
```

I was all super excited and started to trace the path that started spelling out `flag`, each time adjusting `al` to the value that it was being compared to (in `gdb`, this can be done by executing `set $al=0x66`). However, this path spelled out `flagstartswith9`. In other words, I fell for the red herring. D'oh! The other code path started comparing the input to `9`, so I restarted the binary and entered `9447` as the input. Re-tracing the check-input function, I noticed that the code had changed!


```
# Input 'bleh':
   0x7ffff7ff5022:	cmp    al,0x66	
# Input '9447'
   0x7ffff7ff5022:	cmp    al,0x34
```

Very fancy. I traced the function further, ending up here:

```
gdb-peda$ 
[----------------------------------registers-----------------------------------]
RAX: 0x72 ('r')
...
[-------------------------------------code-------------------------------------]
   0x7ffff7ff5062:	movzx  eax,BYTE PTR [rax]
   0x7ffff7ff5065:	movsx  eax,al
   0x7ffff7ff5068:	add    eax,0x39
=> 0x7ffff7ff506b:	cmp    edx,eax
```

This is the fifth character of the password and seems to be `r`. I did a quick `set $edx=$eax` and moved on. The next bytes were `oll`, so I expected the following check to be for `i`. However, the password function borked, because it was using the first four characters to generate the next four! I had only entered four in total. The name of the binary, `rolling`, makes a bit more sense now :)

```
# grab eight input byte
   0x7ffff7ff50c1:	mov    rax,QWORD PTR [rbp-0x8]
   0x7ffff7ff50c5:	add    rax,0x7
   0x7ffff7ff50c9:	movzx  eax,BYTE PTR [rax]
=> 0x7ffff7ff50cc:	movsx  eax,al
# grab third input byte...
   0x7ffff7ff50cf:	mov    rdx,QWORD PTR [rbp-0x8]
   0x7ffff7ff50d3:	add    rdx,0x3
   0x7ffff7ff50d7:	movzx  edx,BYTE PTR [rdx]
   0x7ffff7ff50da:	movsx  edx,dl
# ... and add 0x35 to that third byte!
   0x7ffff7ff50dd:	add    edx,0x35
# compare [3]+0x35 to [7]:
   0x7ffff7ff50e0:	cmp    eax,edx
```

This meant I just had to re-run the binary once I had four more characters. No problem! Eventually, at each `cmp` execution, I noted the proper byte and the correct input turned out to be `9447rollingisfun`. 

The flag was `9447{9447rollingisfun}`. 



