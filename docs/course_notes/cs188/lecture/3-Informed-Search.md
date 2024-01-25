# Informed Search

!!! tip
    If we have some notion of the direction in which we
    should focus our search, we can significantly improve
    performance and "hone in" on a goal much more quickly.
    This is the exactly the focus of **informed search**.

## Heuristics

Heuristics are the driving force that allow estimation of
distance to goal states - they're functions that take in a
state as input and output a corresponding estimate.

## Greedy Search

* *Description*: Always selects the frontier node with the
  *lowest heuristic value* for expansion, which corresponds
  to the state it believes is nearest to the goal.
* *Frontier Representation*: priority queue
* *Completeness and Optimality*: Greedy search is neither
  complete nor optimal.

![img](../img/greedy-search.png){width="700"}

## A* Search

* *Description*: Always selects the frontier node with the
  *lowest estimated total cost* for expansion
    * A* combines the total backward cost (sum of edge
      weights in the path to the state) used by UCS
      with the estimated forward cost (heuristic value)
      used by greedy search by summing them together,
      effectively yielding an $estimated \ \ total \ \ cost$
      from start to goal.
* *Frontier Representation*: priority queue
* *Completeness and Optimality*: A* is both complete and
  optimal given an appropriate heuristic.

## Admissibility and Consistency

!!! question "What makes a heuristic good?"

* $g(n)$ - The function representing total backwards cost
  computed by UCS.

## Dominance

## Search: Summary