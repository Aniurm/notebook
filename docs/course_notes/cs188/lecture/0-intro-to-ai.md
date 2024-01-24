# Intro to AI

## Agents

In AI, the central problem at hand is that of the creation of a **rational agent**,
an entity that
has goals or preferences and tries to perform a series of
**actions** that yield the best/optimal expected outcome
given these goals.

* Reflex agent: doesn’t think about the consequences of its actions
* Planning agent: maintain a model of the world and use this model to simulate performing various actions. Then, the
agent can determine hypothesized consequences of the actions and can select the best one.

## Environment

The
design
of an agent heavily depends on the type of environment the agents acts upon

To define the task environment we use the PEAS (Performance Measure, Environment, Actuators, Sensors) description.

* *Partially observable* environment: the agent does not have full information about the state and
thus the agent must have an internal estimate of the state of the world.
  * contrast to *fully observable* environment: the agent has full access to the state of the world.
* *Stochastic* environment: have uncertainty in the transition model, i.e. taking an action in a specific
state may have multiple possible outcomes with different probabilities.
  * contrast to *deterministic* environment
* *multi-agent* environment: the agent might need to randomize its actions in order to avoid being “predictable" by other agents.
* If the environment does not change as the agent acts on it, then this environment is called *static*. This
is contrast to *dynamic* environments that change as the agent interacts with it.
* *known physics*, *unknown physics*.
