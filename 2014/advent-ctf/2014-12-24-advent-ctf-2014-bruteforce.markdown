### Solved by barrebas

Bruteforce they said, it'll be fun, they said...

We're given only a binary and are told that we shouldn't bruteforce the server. The binary, when started, only says "calculating....." and not much else. Upon closer examination, I found that it does some calculations and checks a certain number before printing out the flag:


```
   0x400703:    movsxd rax,DWORD PTR [rsp+0x8]
=> 0x400708:    cmp    rax,QWORD PTR [rip+0x200969]        # 0x601078
   0x40070f:    je     0x400780
   0x400711:    add    DWORD PTR [rsp+0xc],0x1
```

If `eax` matches the value at `0x601078`, then the code jumps here:

```
  400780:   8b 54 24 0c             mov    edx,DWORD PTR [rsp+0xc]
  400784:   be b1 09 40 00          mov    esi,0x4009b1 ; bruteforce : 0x4009b1 ("the flag is: ADCTF_%d\n")
  400789:   bf 01 00 00 00          mov    edi,0x1
  40078e:   31 c0                   xor    eax,eax
  400790:   e8 7b fe ff ff          call   400610 <__printf_chk@plt>
```

So the calculates until a certain value is found and then dumps the flag. I found a couple of rate-limiting things, such as these syscalls:

```
  4008e0:   49 89 ce                mov    r14,rcx
  4008e3:   48 89 fa                mov    rdx,rdi
  4008e6:   4c 89 d7                mov    rdi,r10
  4008e9:   4c 89 ce                mov    rsi,r9
  4008ec:   48 31 c0                xor    rax,rax
  4008ef:   b0 23                   mov    al,0x23  ; nanosleep
  4008f1:   0f 05                   syscall 
```

I didn't want to slow it down so I nop'ed out three of those syscalls, along with the calls to putchar and printf. I ran the binary, occasionaly checking at which it was... but it still was very slow! Time for a different approach...

Running the binary and breaking at the comparison at `0x400708`, I compared the value at `rsp+0x8` and `rsp+0xc` (which is used to print out the flag eventually). I noticed these numbers:

```
rsp+0x8     rsp+0xc
-------------------
    1           1
    2           2
    3           5
    4           7
    5           11
```

It didn't take me long to realize we're looking at prime numbers here. This binary bruteforces prime numbers and prints out the prime number when the comparison at `0x400708` is true. `eax` contains the ordinal number of the last prime found and is compared to `0x989680`. That would be 10,000,000 in decimal. I quickly located a list of [prime numbers](https://primes.utm.edu/lists/small/millions/) and found the 10th million: 179,424,673. 

Therefore, the flag was: `ADCTF_179424673`. 

