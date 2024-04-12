# BN: Independence

## D-Separation

**A node is conditionally independent of all its ancestor nodes in the graph given all of its parents.**

### Causal Chains

![alt text](../img/casual-chains.png){width=100%}

Figure 1 is a configuration of three nodes known as a **causal chain**.

$$
P(x, y, z)=P(z \mid y) P(y \mid x) P(x)
$$

X and Z are not guaranteed to be independent.

However, we can make the statement that $X \newcommand{\indep}{\perp \!\!\! \perp} \indep Z \mid Y$.

$$
\begin{aligned}
P(X \mid Z, y) & =\frac{P(X, Z, y)}{P(Z, y)}=\frac{P(Z \mid y) P(y \mid X) P(X)}{\sum_x P(X, y, Z)}=\frac{P(Z \mid y) P(y \mid X) P(X)}{P(Z \mid y) \sum_x P(y \mid x) P(x)} \\
& =\frac{P(y \mid X) P(X)}{\sum_x P(y \mid x) P(x)}=\frac{P(y \mid X) P(X)}{P(y)}=P(X \mid y)
\end{aligned}
$$

### Common Cause

![alt text](../img/common-cause.png){width=100%}

$$
P(x, y, z)=P(x \mid y) P(z \mid y) P(y)
$$

X is not guaranteed to be independent of Z.

$X \indep Z \mid Y$: X and Z are independent if Y is observed.

$$
P(X \mid Z, y)=\frac{P(X, Z, y)}{P(Z, y)}=\frac{P(X \mid y) P(Z \mid y) P(y)}{P(Z \mid y) P(y)}=P(X \mid y)
$$

### Common Effect

![alt text](../img/common-effect.png){width=100%}

$$
P(x, y, z)=P(y \mid x, z) P(x) P(z)
$$

In the configuration shown in Figure 5, X and Z are independent: $X \indep Z$

However, they are not necessarily independent when conditioned on Y.

Example:

$$
\begin{aligned}
& P(X=\text { true })=P(X=\text { false })=0.5 \\
& P(Z=\text { true })=P(Z=\text { false })=0.5
\end{aligned}
$$

and $Y$ is determined by whether $X$ and $Z$ have the same value:

$$
P(Y \mid X, Z)= \begin{cases}1 & \text { if } X=Z \text { and } Y=\text { true } \\ 1 & \text { if } X \neq Z \text { and } Y=\text { false } \\ 0 & \text { else }\end{cases}
$$

Then X and Z are independent if Y is unobserved. But if Y is observed, then knowing X tells you about Z. So X and Z are
not conditionally independent given Y.

This same logic applies when conditioning on descendants of Y in the graph. If one of Y’s descendant nodes is observed, as in Figure 7, X and Z are not guaranteed to be independent.

![alt text](../img/effect-child.png){width=100%}

### General Case, and D-Separation

We formulate the problem as follows:

!!! question "Problem"
    Given a Bayes Net $G$, two nodes $X$ and $Y$, and a (possibly empty) set of observed nodes
    $\left \{ Z_1, \dots, Z_k \right \}$, must the following statement be true:
    $X \indep Y \mid \left \{ Z_1, \dots, Z_k \right \}$?
    
**D-Separation**(Directed Separation): If a set of variables $Z_1, \dots, Z_k$ d-separates $X$ and $Y$, then $X \indep Y \mid \left \{ Z_1, \dots, Z_k \right \}$. in all possible distributions that can be encoded by the Bayes Net.

!!! note "D-Separation Algorithm"
    1. Shade all observed nodes $\left\{Z_1, \ldots, Z_k\right\}$ in the graph.
    2. Enumerate all undirected paths from $X$ to $Y$.
    3. For each path:
        1. Decompose the path into triples (segments of 3 nodes).
        2. If **all triples are active**, this path is active and *d-connects* $X$ to $Y$.
    4. If **no path** d-connects $X$ and $Y$, then $X$ and $Y$ are d-separated, so they are conditionally independent given $\left\{Z_1, \ldots, Z_k\right\}$

Any path in a graph from X to Y can be decomposed into a set of 3 consecutive nodes and 2 edges - each of which is called a triple.

A triple is active or inactive depending on whether or not the middle node is observed.

![alt text](../img/active-triple.png){width=100%}

### Examples

![alt text](../img/d-exam1.png){width=100%}

![alt text](../img/d-exam2.png){width=100%}

![alt text](../img/d-exam3.png){width=100%}