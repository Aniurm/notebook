# BN: Representation

只看note的话太难懂了。。。。于是看了Lecture

Fall 2018的老师讲得不错，可以多听听

 Lecture一开始没有直接讲BN，而是先铺垫一些概率论

## Conditional Independence

![alt text](../img/dentist.png){width=100%}

这里举了牙痛的例子，有三个变量：Tootheache, Cavity, Catch.

如果我们知道了Cavity，那么Tootheache和Catch就是条件独立的。

* X is conditionally independent of Y given Z: $X \newcommand{\indep}{\perp \!\!\! \perp} \indep Y \mid Z$

if and only if:

$$
\forall x, y, z: P(x, y \mid z)=P(x \mid z) P(y \mid z)
$$

or, equivalently, if and only if

$$
\forall x, y, z: P(x \mid z, y)=P(x \mid z)
$$

总之，unconditional (absolute) independence is very rare.

*Conditional independence* is our most basic and robust form of knowledge about
uncertain environments.

## Bayesian Network Representation

We formally define a Bayes Net as consisting of:

* A directed acyclic graph of nodes, one per variable $X$.
* A conditional distribution for each node $P(X \mid A_1 \dots A_n)$, where $A_i$ is the $i^{th}$ parent of $X$, 

## Structure of Bayes Nets