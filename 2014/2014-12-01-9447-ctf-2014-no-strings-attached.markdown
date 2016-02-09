### Solved by bitvijays, with special thanks to barrebas

In this challenge you are provided with a 32 bit binary, which when executed asks for a password:
```
root@kali:~# ./no_strings_attached 
Welcome to cyber malware control software.
Currently tracking 742475483 bots worldwide
Please enter authentication details:
```

If you enter the wrong details: Access Denied message is displayed.

First step is to solve this is to decompile this binary using <a href="http://decompiler.fit.vutbr.cz/decompilation-run/">Online Decompiler</a>.

If you see the below decompiled code, main function calls the authenticate function, which has wcscmp() function (which compares two strings). wcscmp is a wide character byte version of strcmp. If you further check the next line, it checks for variable g1 value based on which program prints Success or Access Denied.



``` C
/* -------- Function Prototypes --------- */
//Showing only authenticate and main functions
int32_t banner(void);
int32_t prompt_authentication(void);
int32_t decrypt(int32_t * a1, int32_t * a2);
int32_t authenticate(void);

int32_t authenticate(void) {
    int32_t * v1 = (int32_t *)decrypt((int32_t *)"\x3A\x14", (int32_t *)"\x01\x14"); // bp-16
    g1 = 3;
    fgetws();
    int32_t result; // 0x804879c
    if (g1 == 0) {
        // 0x804879c
        result = *v1;
        free((int32_t *)result);
        return result;
    }
    // 0x804874b
    int8_t * wstr;
    wcslen(wstr);
    g1 = *v1;
    wcscmp();
    if (g1 == 0) {
        // 0x8048780
        g1 = 0x8048b44;
        wprintf("Success! Welcome back!\n");
        // branch -> 0x804879c
    } else {
        // 0x804878f
        g1 = 0x8048ba4;
        wprintf("Access denied!\n");
        // branch -> 0x804879c
    }
    // 0x804879c
    result = *v1;
    free((int32_t *)result);
    return result;
}

int main(int argc, char ** argv) {
    // bb
    setlocale(LC_ALL, "");
    banner();
    prompt_authentication();
    authenticate();
    return 0;
}
```

Next, if we use gdb-peda to debug this binary and put a breakpoint on wcscmp function.
```
root@kali:~# gdb -q ./no_strings_attached 
Reading symbols from /root/no_strings_attached...(no debugging symbols found)...done.
gdb-peda$ pdisass authenticate
Dump of assembler code for function authenticate:
**snip**
   0x0804876a <+98>:	mov    DWORD PTR [esp+0x4],eax
   0x0804876e <+102>:	lea    eax,[ebp-0x800c]
   0x08048774 <+108>:	mov    DWORD PTR [esp],eax
   0x08048777 <+111>:	call   0x80484d0 <wcscmp@plt>
   0x0804877c <+116>:	test   eax,eax
   0x0804879f <+151>:	mov    DWORD PTR [esp],eax
   0x080487a2 <+154>:	call   0x8048480 <free@plt>
   0x080487a7 <+159>:	leave  
   0x080487a8 <+160>:	ret    
End of assembler dump.
gdb-peda$ br *authenticate + 111
Breakpoint 1 at 0x8048777
gdb-peda$ 
```
When the control reaches the breakpoint, gdb-peda guesses the two arguments arg[0] and arg[1] for wcscmp function. As, we typed the password hello, we can see that arg[1] must be the original password. If we extract the memory by command "dump memory file_name start_address end_address" we would be able to get the password which is the flag.
```
gdb-peda$ run
Welcome to cyber malware control software.
Currently tracking 2102520441 bots worldwide
Please enter authentication details: hello
[----------------------------------registers-----------------------------------]
EAX: 0xbfff74bc --> 0x68 ('h')
EBX: 0xb7fc1ff4 --> 0x160d7c 
ECX: 0xbfff74bc --> 0x68 ('h')
EDX: 0x3 
ESI: 0x0 
EDI: 0x0 
EBP: 0xbffff4c8 --> 0xbffff4e8 --> 0xbffff568 --> 0x0 
ESP: 0xbfff74a0 --> 0xbfff74bc --> 0x68 ('h')
EIP: 0x8048777 (<authenticate+111>:	call   0x80484d0 <wcscmp@plt>)
EFLAGS: 0x206 (carry PARITY adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804876a <authenticate+98>:	mov    DWORD PTR [esp+0x4],eax
   0x804876e <authenticate+102>:	lea    eax,[ebp-0x800c]
   0x8048774 <authenticate+108>:	mov    DWORD PTR [esp],eax
=> 0x8048777 <authenticate+111>:	call   0x80484d0 <wcscmp@plt>
   0x804877c <authenticate+116>:	test   eax,eax
   0x804877e <authenticate+118>:	jne    0x804878f <authenticate+135>
   0x8048780 <authenticate+120>:	mov    eax,0x8048b44
   0x8048785 <authenticate+125>:	mov    DWORD PTR [esp],eax
Guessed arguments:
arg[0]: 0xbfff74bc --> 0x68 ('h')
arg[1]: 0x804cba0 --> 0x39 ('9')
[------------------------------------stack-------------------------------------]
0000| 0xbfff74a0 --> 0xbfff74bc --> 0x68 ('h')
0004| 0xbfff74a4 --> 0x804cba0 --> 0x39 ('9')
0008| 0xbfff74a8 --> 0xb7fc2440 --> 0xfbad2288 
0012| 0xbfff74ac --> 0x0 
0016| 0xbfff74b0 --> 0x0 
0020| 0xbfff74b4 --> 0x0 
0024| 0xbfff74b8 --> 0x0 
0028| 0xbfff74bc --> 0x68 ('h')
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x08048777 in authenticate ()
gdb-peda$ dump memory file 0x804cba0 0x805cba0
gdb-peda$ cat file
9447{you_are_an_international_mystery}gdb-peda$
```

**The flag is 9447{you_are_an_international_mystery}**.
