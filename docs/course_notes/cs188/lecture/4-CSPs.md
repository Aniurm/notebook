# Constraint Satisfaction Problems

CSPs are a type of **identification problem**, problems in which
we must simply identify whether a state is a goal state or not,
with no regard to how we arrive at that goal.

CSPs are defined by three factors:

1. **Variables**: $X_1, X_2, \ldots, X_n$ 
2. **Domains**: A set of $\{x_1, x_2, \ldots, x_d\}$ representing all possible values 
    that a CSP variable can take on.
3. **Constraints**: Defines restrictions on the values of variables.

> Constraints satisfaction problems are NP-Hard.

We can often get around this by formulating CSPs as search problems,
defining states as **partial assignments**.

### Constraint Graphs

> Another CSP example: map coloring.
>
> Color a map such that no two adjacent regions have the same color.

![map coloring](../img/map-coloring.png){width="400"}

Constraint satisfaction problems are often represented as **constraint graphs**,
where nodes represent variables and edges represent constraints between them.

* *Unary constraints*: involve a single variable in the CSP.
* *Binary constraints*: involve two variables in the CSP.
* *Higher-order constraints*: involve more than two variables.

The value of constraint graphs is that we can use them to extract valuable
information about the structure of the CSPs we are solving.

## Solving Constraint Satisfaction Problems

**Backtracking search**, an optimization on depth-first search, is used
specifically for the problem of constraint satisfaction, with
improvements coming from two main principles:

1. Fix an ordering for variables, and select values for variables in this order.
2. When selecting values for a variable, only select values that
    don't conflict with any previously assigned variables.

![backtracking](../img/brutal-vs-backtracking.png){width="700"}

Though backtracking search is a vast improvement over the brute-forcing of depth first search, we can get more gains in speed still with further improvements through filtering, variable/value ordering, and structural explotation.

## Filtering

!!! tip ""
    Checks if we can **prune the domains of unassigned variables ahead of time** by removing
    values we know will result in backtracking.

### Naive approach: Forward Checking

Whenever a value is assigned to a variable $X_i$, prunes the domains of
unassigned variables that share a constraint with $X_i$ that would
violate the constraint if assigned.

The idea of forward checking can be generalized to **arc consistency**.

### Arc Consistency