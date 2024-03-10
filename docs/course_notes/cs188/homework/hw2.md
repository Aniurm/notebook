# HW2

## Q1 Campus Layout

此题结合一个实际例子考察constraint, arc consistency等问题。
下面是涉及到的知识点：

Unary constraints are constraints that involve a single variable.

After an arc is enforced, **every value in the domain of the tail has at least one possible value in the domain of the head such that the two values satisfy all constraints with each other**. If a value in the tail's domain does not satisfy this, that value can be removed.

When a domain changes after enforcing an arc, all arcs that end at that variable must be re-added to the queue.

## Q2 CSP Properties

### Q2.1

* Even when using arc consistency, backtracking might be needed to solve a CSP.
  * Arc consistency is often not sufficient on its own to solve a CSP.
  * Consider a CSP with three variables, where the only constraint involves all three variables. Because it is not a binary constraint, arc consistency will not reduce any of the domains, and normal backtracking search is necessary.
* Even when using forward checking, backtracking might be needed to solve a CSP.

可以举一个例子说明:

Explicitly, consider a CSP with variables $A \in\{1,2\}, B \in\{1,2\}, C \in\{3,4\}$. The only constraint is that $A+B>C$. Using MRV and LCV, and breaking ties by choosing the lowest value/variable, the first assignment would be $A=1$, while the only solution is $A=2, B=$ $2, C=3$.

### Q2.2

For a CSP with binary constraints that has no solution, some initial values may still pass arc consistency before any variable is assigned.

Example: Three variables, $A, B, C$, each with domains $\{1,2\}$. The only constraint is that no two variables can have the same value. Enforcing arc consistency will not eliminate any value from any domain, because arc consistency only considers two variables at a time.

## Q3 4-Queens

Solve the 4-queens problem using the min-conflicts algorithm.

## Q4 Arc Consistency

题如其名，非常简单的推导。

## Q5 Arc Consistency Properties

In general, to determine whether the CSP has a solution, enforcing arc consistency alone is not sufficient; backtracking may be required.

## Q6 Backtracking Arc Consistency