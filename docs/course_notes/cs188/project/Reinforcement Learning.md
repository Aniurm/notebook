# Project 3: Reinforcement Learning

## Question 1 (5 points): Value Iteration

根据The Bellman Equation，实现Value Iteration算法。

```python
    def runValueIteration(self):
        for _ in range(self.iterations):
            new_values = util.Counter()
            for state in self.mdp.getStates():
                if self.mdp.isTerminal(state):
                    new_values[state] = 0
                else:
                    max_q_value = float('-inf')
                    for action in self.mdp.getPossibleActions(state):
                        q_value = self.computeQValueFromValues(state, action)
                        max_q_value = max(max_q_value, q_value)
                    new_values[state] = max_q_value
            self.values = new_values
```

`util.Counter()`是dictionary，存储state-value pairs。

说好的iterate直到收敛呢？这里指定了iteration的次数，大概有
几个原因：节省计算资源、时间。

```python
    def computeQValueFromValues(self, state, action):
        """
          Compute the Q-value of action in state from the
          value function stored in self.values.
        """
        q_value = 0
        for next_state, prob in self.mdp.getTransitionStatesAndProbs(state, action):
            reward = self.mdp.getReward(state, action, next_state)
            q_value += prob * (reward + self.discount * self.values[next_state])
        return q_value

    def computeActionFromValues(self, state):
        """
          The policy is the best action in the given state
          according to the values currently stored in self.values.

          You may break ties any way you see fit.  Note that if
          there are no legal actions, which is the case at the
          terminal state, you should return None.
        """
        if self.mdp.isTerminal(state):
            return None
        best_action = None
        best_q_value = float('-inf')
        for action in self.mdp.getPossibleActions(state):
            q_value = self.computeQValueFromValues(state, action)
            if q_value > best_q_value:
                best_q_value = q_value
                best_action = action
        return best_action
```

从数学公式就可以理解`computeQValueFromValues`和`computeActionFromValues`的实现。

## Question 2 (1 point): Bridge Crossing Analysis
## Question 3 (6 points): Policies
## Question 4 (1 points): Prioritized Sweeping Value Iteration[Extra Credit]
## Question 5 (5 points): Q-Learning
## Question 6 (2 points): Epsilon Greedy
## Question 7 (1 point): Bridge Crossing Revisited
## Question 8 (1 point): Q-Learning and Pacman
## Question 9 (4 points): Approximate Q-Learning