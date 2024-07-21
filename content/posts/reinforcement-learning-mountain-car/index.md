---
title: "Train an AI Agent to play Mountain Car with Reinforcement Learning"
date: 2024-07-10T22:06:27+08:00
draft: false
author: "Yu-Kai \"Steven\" Wang"
tags: ["reinforcement learning", "MDP", "gymnasium"]
categories: ["machine learning"]
featuredImage: '/images/mountain_car.png'
---

A few weeks ago I was chatting with a friend who is just getting into reinforcement learning. 
He asked me for some resources to help him learn better, so naturally I pointed him to the classic RL playground [Gymnasium](https://gymnasium.farama.org/index.html) (formerly known as OpenAI Gym), which I had a lot of fun solving when I first started learning. 

This also reminded me of how rusty I am with the Reinforcement Learning techniques, so I thought about challenging myself to complete these puzzles once again.

This post will showcase how to train an AI agent to play the Mountain Car game using Reinforcement Learning.

### Mountain Car

{{< image src="./images/mountain_car.gif" caption="Mountain Car Environment" >}}

I'm starting with the Mountain Car environment here, as it is one of the easiest games from the package and has a dead simple control: move left, no action, or move right. 
You start with a car somewhere between the bottom of the valley, and the goal is to control the car all the way to the right peak where the yellow flag stands.
The tricky part with this game is that the car does not have enough power to accelerate all the way uphill from stop. 
The player will have to "swing" the car back and forth between the valley to gather enough momentum for the climb.

The code below lets you play the game manually using `j` (move left) and `k` (move right) as control.

```python
import gymnasium as gym
from gymnasium.utils.play import play

play(gym.make('MountainCar-v0', render_mode='rgb_array'), 
    keys_to_action={
        'j': 0,
        'k': 2,
    }, 
    noop=1)
```

To train an AI agent to solve this game, we must first model this environment with mathematical terms.

### Markov Decision Process

We'll model the Mountain Car puzzle as a **Markov Decision Process (MDP)**. 
A Markov Decision Process is made up of **States**, **Transition Probability**, **Actions**, **Rewards**, and **Policy**, brief definitions below:
- **States ($S$)**: Current situation of the agent (e.g. position, velocity, etc.). 
- **Actions ($A$)**: Set of all possible actions of the agent, allowing the agent to affect its current state.
- **Transition Probability  ($P(s, a, s^\prime)$)**: The probability of the agent transitioning from state $s$ to state $s^\prime$ by taking action $a$. 
- **Rewards ($R(s, a, s^\prime)$)**: The cost/gain for taking an action $a$ from state $s$ and ending up in state $s^\prime$.
- **Policy ($\pi(s)$)**: The "decision-making" algorithm that decides what action to take given a specific state.

{{< admonition type=note open=true >}}
For a more detailed explanation of MDP, check out this [guide](https://gibberblot.github.io/rl-notes/single-agent/MDPs.html#markov-decision-processes).
{{< /admonition >}}

At the core of the Markov Decision Process is the **Markov Property**. The **Markov Property** states that the next state of the agent only depends on its current state, irrelevant to any historical events. 
This implies that the AI agent should be able to predict the optimal action to take based on only the current state of the agent (therefore, the optimal policy could be described as $\pi(s)$). 

{{< image src="./images/mdp.png" caption="Visualization of an MDP (reward is ignored), states and actions are represented as squares and ovals respectively." >}}

Here is an example MDP of the average life of a college student.

We can visualize an MDP by drawing out the states, available actions, and the probability of landing in another state after taking an action. Using the sample MDP in the figure above, we see that as a broke college student, a trip to casino gives you a 1% chance of being filthy rich, 29% of evening out (still a broke college student), and a whopping 70% chance of being homeless and broke.

Jokes aside, to "solve" the MDP, thereby finding the optimal policy $\pi(s)$, we need to utilize the *Bellman Equation*.

{{< admonition type=note title="Implementing a Markov Decision Process in Code" open=true >}}
To implement MDP in code, we'll usually start with creating the *states* of the MDP. In the case of Mountain Car, there are two observable values from the environment: horizontal position $x$, and car velocity $v$. 
An easy way to do this in Python would to divide $x$ and $v$ into multiple "bins", and create a state for all possible combinations of $x$ and $v$.

```python
# bin observation space into MDP states
self.x_space = np.linspace(-1.2, 0.6, x_bin + 1)
self.vel_space = np.linspace(-0.07, 0.07, vel_bin + 1)
# initialize MDP states
self.A = [0, 1, 2]
self.S = [] # MDP states
prev_x, prev_v = self.x_space[0], self.vel_space[0]
for x in self.x_space[1:]:
    for v in self.vel_space[1:]:
        self.S.append({
            'x_range': (prev_x, x),
            'vel_range': (prev_v, v),
            'p': {a:{} for a in self.A},
            'v': 0
        })
        prev_v = v
    prev_x = x
```

The code above is an excerpt of the full Python implementation located at the bottom of the article. 
It first bins `x` and `v` into several intervals and then creates a list of all possible combinations of the `x` and `v` intervals as states. 
The resulting list of states looks something like this:

```txt
[{'x_range': (-1.2, -1.0999999999999999),
  'vel_range': (-0.07, -0.05600000000000001)},
 {'x_range': (-1.2, -1.0999999999999999),
  'vel_range': (-0.05600000000000001, -0.042)},
 {'x_range': (-1.2, -1.0999999999999999),
  'vel_range': (-0.042, -0.027999999999999997)},
 {'x_range': (-1.2, -1.0999999999999999),
  'vel_range': (-0.027999999999999997, -0.013999999999999999)},
...
 {'x_range': (0.5, 0.6),
  'vel_range': (0.05600000000000002, 0.07)}]
```

Next, we have to compute the transition probability $P$ and the corresponding rewards $R$.
The most straightforward method is to use the Monte-Carlo method, where we try out all possible actions on all possible states, and record the corresponding destination states and their rewards. Code excerpt below.

```python
def _build_mdp(self, trials_per_state_action=100):
    '''
    Build an MDP model for the Mountain Car environment using Monte-Carlo simultions.
    '''
    env = gym.make('MountainCar-v0')
    # for each state action, do N trials to compute transition probability
    for s in tqdm(self.S):
        for a in self.A:
            for _ in range(trials_per_state_action):
                env.reset()
                x_init = np.random.uniform(low=s['x_range'][0], high=s['x_range'][1])
                vel_init = np.random.uniform(low=s['vel_range'][0], high=s['vel_range'][1])
                # manually set env state
                env.unwrapped.state = (x_init, vel_init)
                # step
                (dest_x, dest_v), r, isDone, _, _ = env.step(a)
                # give reward if done
                if isDone:
                    r = self.complete_reward
                # find destination state after stepping
                _, dest_idx = self.get_state(dest_x, dest_v)
                if s['p'][a].get(dest_idx) is None:
                    s['p'][a][dest_idx] = (1, r)
                else:
                    s['p'][a][dest_idx] = (s['p'][a][dest_idx][0] + 1, s['p'][a][dest_idx][1] + r)
```

Again, this might not make sense at the moment, see the full implementation in python at the end of the article and try to run it yourself.

{{< /admonition >}}

### Bellman Equation

The [**Bellman Equation**](https://en.wikipedia.org/wiki/Bellman_equation) states that the expected long-term reward of an action is equal to the reward of the current action plus the expected reward to be earned from future actions from this time on. 
With this idea in mind, we can define a **Value Function $V(s)$**, which is the expected future reward from the current state $s$. 

$$
V(s) = \mathbb{E}\[R_{t+1} + \gamma V(S_{t+1})\]
$$

Where $R_{t+1}$ denotes the reward of going to the next state, and $S_{t+1}$ is the next state. Notice how the value function is recursive. The $\gamma$ term here is the *discount factor* (value between 0 and 1) since rewards further away in the future are less important, and also so that this infinite series converges.

With this value function, the optimal policy would then be picking the action that gives the highest expected rewards, given the current state. Now all that's left is to solve for the optimal value function $V^*$.

### Value Iteration

**Value Iteration** is a *value-based* method to solve an MDP by finding the optimal value function $V^*$ using dynamic programming. It essentially runs the Bellman equation iteratively until the value converges. Algorithm below.

{{< admonition type=note title="Value Iteration Algorithm" open=true >}}
**set** $V(s) = 0$ **for all** $s \in S$\
**repeat**\
&nbsp;&nbsp;&nbsp;&nbsp; $\delta \leftarrow 0$\
&nbsp;&nbsp;&nbsp;&nbsp; **for each** $s \in S$\
&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; $V^\prime(s) \leftarrow max_{a \in A} \sum_{s^\prime \in S} P(s^\prime, a,s)\[r(s,a,s^\prime) + \gamma V(s^\prime)\]$\
&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; $\delta \leftarrow max(\delta, |V^\prime (s) V(s)|)$\
&nbsp;&nbsp;&nbsp;&nbsp; $V \leftarrow V^\prime$\
**until** $\delta \leq \theta$
{{< /admonition >}}

Usually when we are implementing this in code, we store all possible values of $V^\prime(s)$ for each state-actions pair in a table $Q(s, a)$, also known as the *Q-value table*.
This is so that we can easily figure out the optimal policy later on without having to recompute all these values.

{{< admonition type=note title="Value Iteration Algorithm (Q-table)" open=true >}}
**set** $Q(s, a) = 0$ **for all** $s \in S$ **for all** $a \in A$\
**repeat**\
&nbsp;&nbsp;&nbsp;&nbsp; $\delta \leftarrow 0$\
&nbsp;&nbsp;&nbsp;&nbsp; **for each** $s \in S$\
&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; **for each** $a \in A$\
&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; $Q(s, a) \leftarrow \sum_{s^\prime \in S} P(s^\prime, a,s)\[r(s,a,s^\prime) + \gamma V(s^\prime)\]$\
&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; $\delta \leftarrow max(\delta, |max_{a \in A} [Q(s, a)] -  V(s)|)$\
&nbsp;&nbsp;&nbsp;&nbsp; $V(s) \leftarrow max_{a \in A} Q(s, a)$\
**until** $\delta \leq \theta$
{{< /admonition >}}

### Policy Extraction

{{< image src="./images/mc-discrete-stacked.gif" caption="Mountain car trained using value iteration, with slightly different initial conditions." >}}

As we have already discussed earlier, the optimal policy at state $s$ is the action $a$ that maximizes $V(s)$. 
If you have implemented the Q-Table approach earlier, it is the equivalent of finding the argmax of the Q-table on the axis of action.

$$
\pi(s) = argmax_{x \in a} Q(s, a)
$$

```python
# find optimal policy pi(s)
np.argmax(mdp.qa_table, axis=1)
# OUTPUT (0: left, 1: no action, 2: right)
# array([0, 0, 0, 2, 2, 2, 2, 2, 2, 1, 2, 1, 1, 2, 2, 2, 2, 2, 2, 2, 1, 2,
#        0, 1, 2, 2, 2, 2, 2, 1, 0, 0, 0, 0, 2, 2, 2, 2, 2, 2, 2, 0, 0, 0,
#        2, 2, 2, 2, 2, 1, 2, 0, 0, 0, 0, 2, 2, 1, 2, 0, 2, 0, 0, 0, 0, 2,
#        2, 2, 2, 2, 2, 0, 0, 0, 0, 0, 2, 2, 2, 2, 1, 0, 0, 0, 0, 0, 2, 2,
#        2, 2, 2, 0, 0, 0, 0, 0, 1, 2, 2, 1, 0, 0, 0, 0, 0, 0, 2, 2, 2, 2,
#        2, 0, 0, 0, 0, 0, 2, 2, 2, 2, 0, 0, 0, 0, 1, 0, 2, 2, 2, 2, 2, 0,
#        0, 0, 0, 0, 2, 2, 2, 2, 2, 0, 0, 0, 0, 2, 2, 2, 2, 2, 2, 0, 0, 0,
#        1, 2, 2, 2, 2, 2, 1, 0, 0, 0, 2, 2, 2, 1, 1, 2, 2, 0, 0, 2, 2, 1,
#        1, 1, 1, 1], dtype=int64)
```

This step is also known as *policy extraction*, where we find the optimal "solution" for our AI agent to take given the current state.

{{< admonition type=note open=true >}}
There is also another branch of *policy-based* algorithms for finding the optimal policy, but I found *value-iteration* easier to implement. Check out this [guide](https://gibberblot.github.io/rl-notes/single-agent/policy-iteration.html#sec-policy-iteration)](https://gibberblot.github.io/rl-notes/single-agent/policy-iteration.html#sec-policy-iteration) for more info.
{{< /admonition >}}

### Python Implementation

Full implementation of the Mountain Car MDP in Python below: 

```python
class MountainCarDiscreteMDP(object):
    def __init__(self, x_bin, vel_bin, trials_per_state_action=100, gamma=0.99, complete_reward=100):
        '''
        Initialize and build an MDP for the Mountain Car environment. 
        `x_bin` and `v_bin` specifies the granularity of the bins for horizontal spaces and velocity. 
        `trials_per_state_action` sets the number of iterations to run the monte-carlo simulations to compute the transition probability.
        `gamma` is the discount factor for the Bellman Equation.
        `complete_reward` is the reward for the AI agent for winning the game (driving the car up the mountain). 
        '''
        # bin observation space into MDP states
        self.x_space = np.linspace(-1.2, 0.6, x_bin)
        self.vel_space = np.linspace(-0.07, 0.07, vel_bin)
        # initialize MDP states
        self.A = [0, 1, 2]
        self.S = [] # MDP states
        prev_x, prev_v = self.x_space[0], self.vel_space[0]
        for x in self.x_space[1:]:
            for v in self.vel_space[1:]:
                self.S.append({
                    'x_range': (prev_x, x),
                    'vel_range': (prev_v, v),
                    'p': {a:{} for a in self.A},
                    'v': 0
                })
                prev_v = v
            prev_x = x
        # estimate transition probability and rewards
        self.trials_per_state_action = trials_per_state_action
        self.gamma = gamma
        self.complete_reward = complete_reward
        self._build_mdp(trials_per_state_action=trials_per_state_action)
    
    def _build_mdp(self, trials_per_state_action=100):
        '''
        Build an MDP model for the Mountain Car environment using Monte-Carlo simultions.
        '''
        env = gym.make('MountainCar-v0')
        # for each state action, do N trials to compute transition probability
        for s in tqdm(self.S):
            for a in self.A:
                for _ in range(trials_per_state_action):
                    env.reset()
                    x_init = np.random.uniform(low=s['x_range'][0], high=s['x_range'][1])
                    vel_init = np.random.uniform(low=s['vel_range'][0], high=s['vel_range'][1])
                    # manually set env state
                    env.unwrapped.state = (x_init, vel_init)
                    # step
                    (dest_x, dest_v), r, isDone, _, _ = env.step(a)
                    # give reward if done
                    if isDone:
                        r = self.complete_reward
                    # find destination state after stepping
                    _, dest_idx = self.get_state(dest_x, dest_v)
                    if s['p'][a].get(dest_idx) is None:
                        s['p'][a][dest_idx] = (1, r)
                    else:
                        s['p'][a][dest_idx] = (s['p'][a][dest_idx][0] + 1, s['p'][a][dest_idx][1] + r)
        env.close()
    
    def play(self):
        '''
        Test play the Mountain Car game manually.
        '''
        play(gym.make('MountainCar-v0', render_mode='rgb_array'), keys_to_action={
            'j': 0,
            'k': 2
        }, noop=1)
                    
    def get_state(self, x, v):
        '''
        Given the current horizontal position x and velocity of the car v, find the corresponding MDP state.
        '''
        a = max(0, np.searchsorted(self.x_space, min(x, self.x_space[-1]), side='left') - 1)
        b = np.searchsorted(self.vel_space, min(v, self.vel_space[-1]), side='left') - 1
        idx = a * (self.vel_space.shape[0] - 1) + b
        if idx >= len(self.S) or idx < 0:
            raise IndexError(f'Index {idx} out of range for x, v of {(x, v)} and a, b of {(a, b)}')
        return self.S[idx], idx
    
    def value_iteration(self, theta=1e-3, max_iter=5000):
        '''
        Find optimal value function V* with value iteration.
        '''
        # initialize Q(s,a) table
        self.qa_table = np.zeros((len(self.S), len(self.A)))
        delta = -1
        idx = 0
        while ((delta == -1 or delta > theta) and idx < max_iter):
            print(f'Running iter {idx}...', end='')
            delta = 0
            for s_id, s in enumerate(self.S):
                for a in self.A:
                    # get transition probability given action
                    sums = 0
                    for dest_idx, (freq, rewards) in s['p'][a].items():
                        p = freq / self.trials_per_state_action
                        r = rewards / self.trials_per_state_action
                        v = self.gamma * self.S[dest_idx]['v']
                        sums += p * (r + v)
                    # update Q-value                    
                    self.qa_table[s_id, a] = sums
                # get new v value and update
                v_prime = np.max(self.qa_table[s_id])
                delta += np.abs(v_prime - s['v'])
                s['v'] = v_prime
            idx += 1
            print(f', delta={delta:.5f}')
        print('\nDone')
    
    def get_optimal_action(self, s_id):
        '''
        pi(s), return optimal action to take given state s.
        '''
        if not hasattr(self, 'qa_table'):
            raise AttributeError('Missing QA table, did you run "value_iteration()"?')
        # get new v value and update
        a = np.argmax(self.qa_table[s_id])
        return a
    
    def solve(self, max_steps=1000, record=False, episodes=1):
        '''
        Let the AI agent play the Mountain Car game using the computed optimal policy. Set `record` to True to record and save as video (will not render a window). Set `episodes` to specify number of sessions for the AI agent to play.
        '''
        if not hasattr(self, 'qa_table'):
            raise AttributeError('Missing QA table, did you run "value_iteration()"?')
        env = gym.make('MountainCar-v0', render_mode='rgb_array' if record else 'human')
        if record:
            # wrap env with video recorder
            env = RecordVideo(env, './vid', episode_trigger=lambda x: True, name_prefix='mc-discrete')
        for _ in range(episodes):
            (x, v), _ = env.reset()
            isDone = False
            idx = 0
            rewards = 0
            while not isDone and idx < max_steps:
                # observe state
                _, s_id = self.get_state(x, v)
                # get optimal action
                a = self.get_optimal_action(s_id)
                # take optimal action
                (x, v), r, isDone, _, _ = env.step(a)
                # give reward if done
                if isDone:
                    r = self.complete_reward
                rewards += r
                idx += 1
                sleep(0.01)
            print(f'total steps: {idx},  total rewards: {rewards:.3f}')
        env.close()
```

To create an instance of the Mountain Car MDP, run the following line.

```python
mdp = MountainCarDiscreteMDP(x_bin=18, vel_bin=10)
```

The `x_bin` and `vel_bin` specify the number of bins to divide the horizontal axis $x$ and the velocity of the car $v$ to build the MDP.
The resulting MDP will have `x_bin * vel_bin` states (18 * 10 = 180 states in this case). 
Setting these numbers higher results in a more fine-grained state definition, but at the cost of memory and time.

To manually play the game, run this.

```python
mdp.play()
```

Note that the game resets once every 100 steps.

To run value iteration, run this.

```python
mdp.value_iteration()
```

You can toy with the `theta` argument to stop the value iteration earlier (or later).

Finally, to let the AI agent trained using value iteration play the game, run this.

```
mdp.solve()
```

That's it! You can now try to solve all the classic control environments in Gymnasium using this same exact framework. 

### References

1. [Value Iteration](https://gibberblot.github.io/rl-notes/single-agent/value-iteration.html)
2. [Gymnasium](https://gymnasium.farama.org/index.html)
3. [Mountain Car](https://gymnasium.farama.org/environments/classic_control/mountain_car/)
4. [Markov Decision Process](https://gibberblot.github.io/rl-notes/single-agent/MDPs.html)