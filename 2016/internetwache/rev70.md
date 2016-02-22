### Solved by superkojiman

It's an ARM binary! The good stuff happens in the handle_task function. I opened up the binary in IDA and found the following: 

```
.text:00010810                 LDR     R1, =aIw        ; "IW{"
.text:00010814                 LDR     R0, =aS         ; "%s"
.text:00010818                 BL      printf
.text:0001081C                 MOV     R0, #'S'        ; c
.text:00010820                 BL      putchar
.text:00010824                 MOV     R2, #'E'
.text:00010828                 MOV     R1, #'.'
.text:0001082C                 LDR     R0, =aCC        ; "%c%c\n"
.text:00010830                 BL      printf
.text:00010834                 B       loc_10918
```

"IW{" looks like the start of the flag. This statement prints out "IW{S.E". If we follow through, we can see that it's a series of if/else conditions within a switch block. We don't need to run the binary, just look at each switch case and the conditions in each one, and from there it's pretty obvious which case prints the correct piece of the flag: 

```
; prints out ".R.V.E"
.text:00010874                 LDR     R1, =a_r_       ; ".R."
.text:00010878                 LDR     R0, =aS         ; "%s"
.text:0001087C                 BL      printf
.text:00010880                 MOV     R0, #'V'        ; c
.text:00010884                 BL      putchar
.text:00010888                 MOV     R2, #'E'
.text:0001088C                 MOV     R1, #'.'
.text:00010890                 LDR     R0, =aCC        ; "%c%c\n"
.text:00010894                 BL      printf

; prints out ".R>=F:"
.text:000108A8                 LDR     R1, =aHereSYour3_Blo ; jumptable 00010760 case 2
.text:000108AC                 LDR     R0, =aS         ; "%s"
.text:000108B0                 BL      printf
.text:000108B4                 LDR     R1, =a1337      ; "1337"
.text:000108B8                 LDR     R0, [R11,#s]    ; s1
.text:000108BC                 BL      strcmp
.text:000108C0                 MOV     R3, R0
.text:000108C4                 CMP     R3, #0
.text:000108C8                 BNE     loc_108D8
.text:000108CC                 LDR     R0, =a_rF       ; ".R>=F:"
.text:000108D0                 BL      puts
.text:000108D4                 B       loc_10918

; prints out "A:R:M}"
.text:000108F0                 LDR     R3, [R11,#s]    ; jumptable 00010760 case 3
.text:000108F4                 LDRB    R3, [R3]
.text:000108F8                 CMP     R3, #0
.text:000108FC                 BEQ     loc_10914
.text:00010900                 MOV     R3, #'}'
.text:00010904                 LDR     R2, =aRM        ; ":R:M"
.text:00010908                 MOV     R1, #'A'
.text:0001090C                 LDR     R0, =aCSC       ; "%c%s%c\n"
.text:00010910                 BL      printf
```

This gives us the flag: IW{S.E.R.V.E.R>=F:A:R:M}
