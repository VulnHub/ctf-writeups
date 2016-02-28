### Solved by superkojiman

This was an easy one. We're given a 64-bit PE executable. A quick examination with radare2 revealed two interesting functions in main():

```
|      |    0x004015dc      e8d7160000     call sym.strcmp
|      |    0x004015e1      85c0           test eax, eax
|      |,=< 0x004015e3      7507           jne 0x4015ec               
|      ||   0x004015e5      e825ffffff     call sym.theflagishere
|     ,===< 0x004015ea      eb05           jmp 0x4015f1               
|     |||   ; JMP XREF from 0x004015e3 (sym.main)
|     ||`-> 0x004015ec      e8fffeffff     call sym.theflagisnothere
|     ||    ; JMP XREF from 0x004015ea (sym.main)
|     ||    ; JMP XREF from 0x0040158d (sym.main)
|     ``--> 0x004015f1      4883c430       add rsp, 0x30
|           0x004015f5      5d             pop rbp
\           0x004015f6      c3             ret
```

theflagishere() function caught my eye, and so I examined it:

```
[0x004014d0]> pdf@sym.theflagishere 
/ (fcn) sym.theflagishere 79
|           ; CALL XREF from 0x004015e5 (sym.theflagishere)
|           0x0040150f      55             push rbp
|           0x00401510      4889e5         mov rbp, rsp
|           0x00401513      4883ec20       sub rsp, 0x20
|           0x00401517      488d0df72a00.  lea rcx, [rip + 0x2af7]     ; 0x404015 ; str.bVas ; "bVas" @ 0x404015
|           0x0040151e      e87d170000     call sym.printf
|           0x00401523      488d0df02a00.  lea rcx, [rip + 0x2af0]     ; 0x40401a  ; "qdf" @ 0x40401a
|           0x0040152a      e871170000     call sym.printf
|           0x0040152f      488d0de82a00.  lea rcx, [rip + 0x2ae8]     ; 0x40401e  ; "Nov" @ 0x40401e
|           0x00401536      e865170000     call sym.printf
|           0x0040153b      488d0de02a00.  lea rcx, [rip + 0x2ae0]     ; 0x404022 ; str.aQnn ; "aQnn" @ 0x404022
|           0x00401542      e859170000     call sym.printf
|           0x00401547      488d0dd92a00.  lea rcx, [rip + 0x2ad9]     ; 0x404027  ; "aww" 0x00404027  ; "aww" @ 0x404027
|           0x0040154e      e84d170000     call sym.printf
|           0x00401553      b800000000     mov eax, 0
|           0x00401558      4883c420       add rsp, 0x20
|           0x0040155c      5d             pop rbp
\           0x0040155d      c3             ret
```

A relatively short function that basically prints the flag out. In fact, radare was kind enough to display the string that was to  be printed; ```bVasqdfNovaQnnaww```, which wass the flag. 
