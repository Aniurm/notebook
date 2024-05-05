# Particle Filtering

!!! note
    The Hidden Markov Model analog to Bayes' net sampling is called **particle filtering**,
    and involves simulating the motion of a set of particles through a state graph
    to approximate the probability (belief) distribution of the random variables
    in question. This solves the same question as the Forward Algorithm: it gives us an approximation of $P\left(X_N \mid e_{1: N}\right)$

Our belief that a particle is in any given state at any given timestep is dependent entirely on the number of particles in that state at that timestep in our simulation.

### Particle Filtering Simulation

* *Particle Initialization*. 
* *Time Elapse Update*: Update the value of each particle according to the transition model.
* *Observation Update*: During the observation update for particle filtering, we use the sensor model $P\left(F_i \mid T_i\right)$ to weight each particle according to the probability dictated by the observed evidence and the particle’s state. Specifically, for a particle in state $t_i$ with sensor reading $f_i$, assign a weight of $P\left(f_i \mid t_i\right)$ to the particle.
    1. Calculate the weights of all particles
    2. Calculate the total weight for each state.
    3. If the sum of all weights across all states is 0, reinitialize all particles.
    4. Else, normalize the distribution of total weights over states and resample your list of particles from this distribution.

## Summary

- *Markov models*, which encode **time-dependent random variables that possess the Markov property**. We can compute a belief distribution at any timestep of our choice for a Markov model using probabilistic inference with the mini-forward algorithm.
- *Hidden Markov Models*, which are **Markov models with the additional property that new evidence which can affect our belief distribution can be observed at each timestep**. To compute the belief distribution at any given timestep with Hidden Markov Models, we use the forward algorithm.

Sometimes, running exact inference on these models can be too computationally expensive, in which case we can use particle filtering as a method of approximate inference.