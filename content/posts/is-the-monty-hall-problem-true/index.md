---
title: "Is The Monty Hall Problem True?"
date: 2024-08-11T13:05:25+08:00
draft: false
author: "Yu-Kai \"Steven\" Wang"
tags: [math, simulation, probability]
categories: [data science]
featuredImage: '/images/three_huge_doors_2.png'
---

You are presented with three doors, and there is a prize hidden behind only one of the doors.
You chose door A, but before you can open the door, the host opens door B to show you that there is nothing behind it. 
Now you are given to chance to choose between sticking with your original plan (door A) or switching to door C. 

*What should you do?*

It's been a while since I last heard of ***The Monty Hall Problem*** until I came across [Zack D. Film's video](https://www.youtube.com/watch?v=xLccN8V8hII) while scrolling through YouTube Shorts a few days ago, and I thought it would be fun to prove this empirically with some good old Monte Carlo simulation. 

Here we go!

### To switch, or not to switch?

**The optimal solution to the Monty Hall Problem, according to statistics, is to always switch.** 
I'm not going to dive deep into the math behind it, but I'll try to explain it in layman's terms: 

1. We start with doors A, B, and C, each with a 1/3 probability of having the prize. 
2. The moment we chose door A to open, the probability then converges to 1/3 behind door A, and 2/3 not behind door A (thus behind B or C). 
3. The host then reveals that door B is empty. 
4. The previous 2/3 probability of the prize sitting behind door B or C then becomes the probability of the prize sitting behind door C, since B is eliminated
5. As a result, switching doors will now increase your probability of picking the right door from 1/3 to 2/3, therefore switching is always recommended.

{{< image src="./images/monty_hall_prob.png" caption="Monty Hall Probability Diagram" >}}

Let's try to validate this theory with some simulation.

### Python Simulation

We'll simulate 10000 such trials using the following script, and calculate the win rate of the two strategies: 
1. Strategy 1: Do not switch doors.
2. Strategy 2: Always switch doors. 

```python
import numpy as np


# experiment iteration
N = 10000

# randomly generate N prizes
games = np.random.randint(low=0, high=3, size=N)

# simulate player random picks
picks = np.random.randint(low=0, high=3, size=N)

# simulate revelation from host
reveal = []
for g, p in zip(games, picks):
    # if pick is the prize, randomly reveal another door
    choices = [0, 1, 2]
    if g == p:
        choices.remove(g)
        reveal.append(choices[np.random.randint(low=0, high=2)])
    else:
        # reveal the only non-empty door left
        choices.remove(g)
        choices.remove(p)
        reveal.append(choices[0])
reveal = np.array(reveal)

# strategy 1: do not switch
strat_1 = np.sum(picks == games) / N

# strategy 2: switch
new_pick = []
for r, p in zip(reveal, picks):
    choices = [0, 1, 2]
    choices.remove(r)
    choices.remove(p)
    new_pick.append(choices[0])
new_pick = np.array(new_pick)
strat_2 = np.sum(new_pick == games) / N

print(f'strategy 1: {strat_1:.2f}')
print(f'strategy 2: {strat_2:.2f}')
```

Here is the output:

```
strategy1 1: 0.33
strategy 2: 0.67
```

Strategy 1 and 2 have a win rate of approximately 1/3 and 2/3, exactly what we are expecting!

### Final Thoughts

The Monty Hall Problem is a carefully crafted probability illusion designed to trick the audience into thinking that the probability doesn't change after new information is introduced (host eliminating one option). 
Using a more obvious scenario could help people better understand why the probability changed:

Assume that instead of three doors, there are now 1,000,000 doors for you to choose from. After choosing a door, the host opened another 999,998 doors with nothing behind it. 
You are now left with two choices: sticking with your original hunch, knowing that it has a 1 in a million chance of winning, or switching doors (which now has a 99.9999% chance of winning)?

Until next time!