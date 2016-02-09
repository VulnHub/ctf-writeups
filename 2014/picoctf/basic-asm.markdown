### Solved by superkojiman

Basic ASM is a 60 point reverse engineering challenge. 

> We found this program snippet.txt, but we're having some trouble figuring it out. What's the value of %eax when the last instruction (the NOP) runs?

The provided snippets.txt shows the following assembly source code: 

```asm
# This file is in AT&T syntax - see http://www.imada.sdu.dk/Courses/DM18/Litteratur/IntelnATT.htm
# and http://en.wikipedia.org/wiki/X86_assembly_language#Syntax. Both gdb and objdump produce
# AT&T syntax by default.
MOV $119,%ebx
MOV $28557,%eax
MOV $8055,%ecx
CMP %eax,%ebx
JL L1
JMP L2
L1:
IMUL %eax,%ebx
ADD %eax,%ebx
MOV %ebx,%eax
SUB %ecx,%eax
JMP L3
L2:
IMUL %eax,%ebx
SUB %eax,%ebx
MOV %ebx,%eax
ADD %ecx,%eax
L3:
NOP
```

So we need to find the value of EAX when the NOP is hit. The hint is to code it out in C, but there's no need for that. Just save the code in basic.s with a few small changes: 

```asm
.global main
.text

main:

MOV $119,%ebx
MOV $28557,%eax
MOV $8055,%ecx
CMP %eax,%ebx
JL L1 
JMP L2 
L1:
IMUL %eax,%ebx
ADD %eax,%ebx
MOV %ebx,%eax
SUB %ecx,%eax
JMP L3 
L2:
IMUL %eax,%ebx
SUB %eax,%ebx
MOV %ebx,%eax
ADD %ecx,%eax
L3:
INT3    # <--- set a breakpoint here
NOP
```

Next compile it with gcc: 

```
# gcc - basic basic.s
```

Run in gdb and get the value of EAX, which is the flag:

```
# gdb ./basic -q -batch -n -ex 'r' -ex 'p $eax'

Program received signal SIGTRAP, Trace/breakpoint trap.
0x08048406 in L3 ()
$1 = 3418785
```

The flag is **3418785**
