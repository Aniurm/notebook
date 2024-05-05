# Hidden Markov Models

## Markov Models

Markov model can be thought of as analogous to a chain-like, infinite-length
Bayes' net.

Our weather model will be time-dependent (as are Markov models in general)

![alt text](../img/weather.png){width=100%}

To track how our quantity under consideration changes over time,
we need to know the following:

* **Initial Distribution**: The probability of the quantity at time 0.
* **Transition Model**: The probability of moving from one state to another between time steps.

The weather at time $t = i + 1$ satisfies the **Markov Property** or 
**memoryless property**, and is independent of the weather at all other 
timesteps besides $t = i$.

Markov models make the following assumptions:

$$
W_{i+1} \newcommand{\indep}{\perp \!\!\! \perp} \indep \left\{W_0, \ldots, W_{i-1}\right\} \mid W_i
$$

This allows us to reconstruct the joint distribution:

$$
P\left(W_0, W_1, \ldots, W_n\right)=P\left(W_0\right) P\left(W_1 \mid W_0\right) P\left(W_2 \mid W_1\right) \ldots P\left(W_n \mid W_{n-1}\right)=P\left(W_0\right) \prod_{i=0}^{n-1} P\left(W_{i+1} \mid W_i\right)
$$

A final assumption is that the transition model is **stationary**, meaning that for all values of $i$ (all time steps):
$P\left(W_{i+1} \mid W_i\right)$ is identical.

### The Mini-Forward Algorithm

By properties of marginalization, we know that

$$
P\left(W_{i+1}\right)=\sum_{w_i} P\left(w_i, W_{i+1}\right)
$$

By the chain rule we can re-express this as follows:

$$
P\left(W_{i+1}\right)=\sum_{w_i} P\left(W_{i+1} \mid w_i\right) P\left(w_i\right)
$$

With this equation, we can iteratively compute the distribution of the weather
at any time step by starting with the initial distribution $P\left(W_0\right)$
and using it to compute $P\left(W_1\right)$, and so on.

### Stationary Distribution

!!! question
    Does the probability of being in a state at a given timestep ever converge?

To solve the problem above, we must compute the **stationary distribution** of the Markov model. 

$$
P\left(W_{t+1}\right)=P\left(W_t\right)=\sum_{w_t} P\left(W_{t+1} \mid w_t\right) P\left(w_t\right)
$$

## Hidden Markov Models

**Hidden Markov Models** allows us to observe some evidence at each time step,
which can potentially affect the belief distribution at each of the states.

![alt text](../img/hidden-weather.png){width=100%}

Unlike vanilla Markov models, we now have two different types of nodes:

* $W_i$: **state variable**.
* $F_i$: **evidence variable**.

$$
\begin{array}{lll}
F_1 & \indep W_0 \mid W_1 & \\
\forall i & =2, \ldots, n ; & W_i \indep \left\{W_0, \ldots, W_{i-2}, F_1, \ldots, F_{i-1}\right\} \mid W_{i-1} \\
\forall i & =2, \ldots, n ; & F_i \indep \left\{W_0, \ldots, W_{i-1}, F_1, \ldots, F_{i-1}\right\} \mid W_i
\end{array}
$$


Hidden Markov Models make the following assumptions:

* The transition model $P\left(W_{i+1} \mid W_i\right)$ is stationary.
* The **sensor model** $P\left(F_i \mid W_i\right)$ is also stationary.

#### Believe Distribution

All evidence $F_1, \ldots, F_i$ is observed up to date:

$$
B\left(W_i\right)=P\left(W_i \mid f_1, \ldots, f_i\right)
$$

Evidence $f_1, \ldots, f_{i-1}$ is observed:

$$
B'\left(W_i\right)=P\left(W_i \mid f_1, \ldots, f_{i-1}\right)
$$

Defining $e_i$ as evidence observed at timestep $i$, you might see the aggregated evidence from timesteps $1 \leq i \leq t$ reexpressed in the form:

$$
e_{1: t}=e_1, \ldots, e_t
$$

### The Forward Algorithm

Relationship between $B\left(W_i\right)$ and $B'\left(W_{i + 1}\right)$:

$$
B^{\prime}\left(W_{i+1}\right)=\sum_{w_i} P\left(W_{i+1} \mid w_i\right) B\left(w_i\right)
$$

Relationship between $B'\left(W_{i+1}\right)$ and $B\left(W_{i+1}\right)$:

$$
B\left(W_{i+1}\right) \propto P\left(f_{i+1} \mid W_{i+1}\right) B^{\prime}\left(W_{i+1}\right)
$$

Combining the two relationships we've just derived yields an iterative algorithm known as the **forward algorithm**, the Hidden Markov Model analog of the mini-forward algorithm from earlier:

$$
B\left(W_{i+1}\right) \propto P\left(f_{i+1} \mid W_{i+1}\right) \sum_{w_i} P\left(W_{i+1} \mid w_i\right) B\left(w_i\right)
$$

The forward algorithm can be thought of as consisting of two distinctive steps:

1. **Time elapse update**: determining $B'\left(W_{i+1}\right)$ from $B\left(W_i\right)$.
2. **Observation update**: determining $B\left(W_{i+1}\right)$ from $B'\left(W_{i+1}\right)$.

### Viterbi Algorithm

!!! question "$\arg \max _{x_{1: N}} P\left(x_{1: N} \mid e_{1: N}\right)$"
    What is the most likely sequence of hidden states the system followed given
    the observed evidence variables so far?

