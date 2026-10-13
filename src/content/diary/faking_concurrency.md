
---  
title: "Faking Concurrency in C"  
date: 2026-10-14
summary: "How to run concurrent tasks in C for a single thread CPU by using a pacer which splits up the actions"
---  

When writing out an application in C,  we can write out different functions to run on different threads to stop the whole program from freezing while waiting for a single function. While this is standard practice sometimes we dont have the necessary resources to run multiple threads at a time. This is why we **fake concurrency** by running functions one after another, which are run with a small buffer. This is called **"Time-Sliced Polling."**

### How does it work?
If a bartender makes different cocktails but they can only make one at a time (single-thread cpu).  Each cocktail has different recipes and takes different amounts of time to make. If customers order 2 cocktails (concurrent tasks), one hard to make and the other easy, then the bartender can get stuck making the complex one while the other has to wait. This means the other customer is frozen waiting for their drink. To prevent this the bartender has rules...

#### Rule 1: Kitchen timer
A kitchen timer rings every few seconds (the fixed-rate loop), Then when the timer rings it forces the bartender to check the next ticket.

#### Rule 2: Task Granularity (How broken down a total task is)
No single action can take forever. If a task is "add lemon, add lime, add ice, add vodka, shake, pour" then doing all of that at once hogs the cocktail section. Instead the barista breaks down every drink into micro-actions that can be finished in a single tick.

Tick 1: Add lemon
Tick 2: Check ticket B, Add Orange
Tick 3: Check ticket C, Add Mint
Tick 4: Back to ticket A, Add lime

#### Rule 3: State Retention (Recipe card / checkbox)
When the bartender comes back to a drink, they might not know what they have already added, this is the same in C. This is why we use a "Checkbox" to measure the current state of each cocktail. In C, that checkbox is called a **State Struct**. After you run a function in the stack, those local variables disappear the moment it returns. So when each time the timer goes off, (and we have to go through and check which recipes need doing), we have a starting point to start off from. 

| Ticks  | 1          | 2           | 3       | 4         | 5       | 6    | 7       |
| ------ | ---------- | ----------- | ------- | --------- | ------- | ---- | ------- |
| DrinkA | Add lemon  | Add Lime    | Add Ice | Add Vodka | Shake   | Pour | Garnish |
| DrinkB | Add Orange | Add Bitters | Garnish |           |         |      |         |
| DrinkC | Add mint   | Add lime    | Add Rum | Muddle    | Garnish |      |         |
### How this looks in code
```c
#include <stdio.h> // For demonstration output
#include <stdbool.h>
#include <stdint.h>
#include "system.h"
#include "pacer.h"

#define PACER_RATE 500

/* RECIPE */
typedef enum {
    DRINK_A_ADD_LEMON,
    DRINK_A_ADD_LIME,
    DRINK_A_ADD_ICE,
    DRINK_A_ADD_VODKA,
    DRINK_A_SHAKE,
    DRINK_A_POUR,
    DRINK_A_GARNISH,
    DRINK_A_DONE
} drink_a_state_t;

static void drink_a_task_init(void) {}

/* THE CHECKBOX */
static void drink_a_task(void) {
    static drink_a_state_t state = DRINK_A_ADD_LEMON;

    switch (state) {
        case DRINK_A_ADD_LEMON:
            printf("[Bartender] Drink A: Added lemon\n");
            state = DRINK_A_ADD_LIME;
            break;
        case DRINK_A_ADD_LIME:
            printf("[Bartender] Drink A: Added lime\n");
            state = DRINK_A_ADD_ICE;
            break;
        case DRINK_A_ADD_ICE:
            printf("[Bartender] Drink A: Added ice\n");
            state = DRINK_A_ADD_VODKA;
            break;
		// Contine on for each task... 
}
/* Recipe for Drink B */
typedef enum {
    DRINK_B_ADD_ORANGE,
    DRINK_B_ADD_BITTERS,
    DRINK_B_GARNISH,
    DRINK_B_DONE
} drink_b_state_t;

static void drink_b_task_init(void) {}

static void drink_b_task(void) {
	/* Repeat similar structure to drink A*/
}


/* Recipe for Drink C */
typedef enum {
    DRINK_C_ADD_MINT,
    DRINK_C_ADD_LIME,
    DRINK_C_ADD_RUM,
    DRINK_C_MUDDLE,
    DRINK_C_GARNISH,
    DRINK_C_DONE
} drink_c_state_t;

static void drink_c_task_init(void) {}

static void drink_c_task(void) {
	/* Repeat similar structure to drink A*/
}


int main (void)
{
    uint16_t counter = 0;
    system_init();

    drink_a_task_init();
    drink_b_task_init();
    drink_c_task_init();
    // This means main will run 500 times a second (500 ticks/s)
    pacer_init (PACER_RATE);
    
    // Each task should fire off at different frequencys from one another,
    // this means they naturally fire off at different times.
	drink_A_tick_rate = 100;
	drink_B_tick_rate = 150;
	drink_C_tick_rate = 200;
	
    while (1)
    {
        if (counter % drink_A_tick == 0) {
            drink_a_task();
        }

        if (counter % drink_B_tick == 0) {
            drink_b_task();
        }

        if (counter % drink_C_tick == 0) {
            drink_c_task();
        }

        counter++;
        pacer_wait ();
    }
    return 0;
}
```

#### What should the tick rate be?

The individual **task** tick rates are decided based of their requirements. For a user inputs, we would use 50-100Hz (10-20ms), as humans find it hard to notice a delay under 50ms.
However the **Pacers** tick rate should be the at least as fast the most demanding periodic task, if your display refreshs at 250Hz, then PACER_RATE cannot be below 250. Ideally the pacer rate that evenly devides into each chosen tick rate.

#### Final Reflections
Learning about the pacer rate was ver interesting and I struggled to get my head around it for a bit. The best way I can remember it is if your octopus that can only move one hand at a time. Octopus needs to do fractions of each task over the total period required to do the longest task.