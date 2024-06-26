# Probability

## Probability Rundown

Here we provide a brief summary of probability rules we will be using.

A **random variable** represents an event whose outcome is uncertain. A **probability distribution** is 
an assignment of weights to outcomes. Probability distributions must satisfy the following properties:

$$
\begin{aligned}
& 0 \leq P(\omega) \leq 1 \\
& \sum_\omega P(\omega)=1
\end{aligned}
$$

We use the notion $P(A, B, C)$ to denote the **joint distribution** of the variables $A$, $B$, $C$. In joint distributions ordering does not matter i.e. $P(A, B, C) = P(C, B, A)$. 

We can expand a joint distribution using the **chain rule**:

$$
\begin{aligned}
& P(A, B)=P(A \mid B) P(B)=P(B \mid A) P(A) \\
& P\left(A_1, A_2 \ldots A_k\right)=P\left(A_1\right) P\left(A_2 \mid A_1\right) \ldots P\left(A_k \mid A_1 \ldots A_{k-1}\right)
\end{aligned}
$$

The **marginal distribution** of $A, B$ can be obtained by summing out all possible values
that variable $C$ can take as $P(A, B) = \sum_c P(A, B, C = c)$.
The marginal distribution of $A$ can also be obtained as $P(A) = \sum_b \sum_c P(A, B = b, C = c)$.

When we do operations on probability distributions, sometimes we get distributions that do not necessarily sum to 1. To fix this, we **normalize**: take the sum of all entries in the distribution and divide each entry by this sum.

**Conditional probabilities** assign probabilities to events conditioned on some known facts.

$$
P(A \mid B)=\frac{P(A, B)}{P(B)}
$$

**Bayes' Rule**:

$$
P(A \mid B)=\frac{P(B \mid A) P(A)}{P(B)}
$$

To write that random variables $A$ and $B$ are **mutually independent**, we write $A  \newcommand{\indep}{\perp \!\!\! \perp} \indep B$, equivalent to $B \indep A$. 

## Probability Inference

For the next several weeks, we will use a new model where each possible state for the world
has its own probability. More precisely, our model is a **joint distribution**, i.e.
a table of probabilities which captures the likelihood of each possible **outcome**,
also known as an **assignment** of variables.

### Inference by Enumeration

Given a joint PDF, we can trivially compute any desired probability distribution $P\left(Q_1 \ldots Q_m \mid e_1 \ldots e_n\right)$ using **inference by enumeration**, for which we define three types of variables:

1. **Query variables** $Q_i$, which are unknown and appear on the left side of the conditional ($\mid$) in the 
   desired probability distribution.
2. **Evidence variables** $e_i$, which are observed variables whose values are known and appear on the right side of
   the conditional ($\mid$) in the desired probability distribution.
3. **Hidden variables**, which are values presents in the overall joint distribution but not in the desired
   distribution.

In Inference By Enumeration, we follow the following algorithm:

1. Collect all the rows consistent with the observed evidence variables.
2. Sum out (marginalize) all the hidden variables.
3. Normalize the table so that it is a probability distribution (i.e. values sum to 1)