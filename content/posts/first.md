---
title: "The name's R... Algorithm R"
subtitle: "An introduction to reservoir sampling"
date: 2023-02-16T00:34:43Z
draft: false
summary: "A reservoir sampling algorithm to randomly draw a sample of size $n$ from a population of unknown size."
---

## Random sampling

Explain with and without replacement. Begin with a visual explanation, only then switch to mathematical details

Motivate random sampling (huge amounts of data, sometimes impossible to get all of it, sometimes all of it is just too much to process, etc)

Consider the problem of random sampling where, from a given population of size $N$, we want to extract a sample of size $n$, with $n < N$, such that not given item from the population was selected with higher probability than the remaining items and that this selection was performed without replacement. 

To solve such a issue, we'd begin by calculating what we will call the **inclusion probability**, $p$, which will be given by the quotient between the sample size and the population size, $p=n/N$. Since both $N$ and $n$ are taken to be larger than zero and $n < N$, we have that $p$ is a proper probability, *i.e.*, it is between 0 and 1. Then, we'd sequentially iterate over our population's items. For each item, we'd generate a random number, $k$, between zero and one. If $k < p$, then we add the item to our sample, otherwise, we discard it. Since this requires a single pass over the entire population, this algorithm trivially runs in $\mathcal{O}(N)$ time.

We could trivially take $n=0$. In other words, we can say that the sample we want to extract from our population contains no samples at all. Fortunately, our math and logic still holds since this would lead to an inclusion probability of zero. 

Talk about $n=0$ as the sample of trivial size, and how our process holds for this case, since during iteration nothing would ever be selected.

Talk about $N=0$ being an ill-defined problem because there is no population. It does not make sense.

While thinking about this case might seem silly, it does serve as an important sanity check. For the situation above the calculation of the inclusion probability is very straightforward and thus there is no confusion or danger that $n=0$ might lead to $p \neq 0$, but this isn't always the case.

{{< admonition type=note title="The Binomial Distribution" open=true >}}
Let $X$ be a discrete random variable that represents the probability of obtaining $k$ successes in a series of $n$ independent trials with replacement, all of which with probability of success $p$. Then, $X$ is said to follow a Binomial distribution with parameters $n$ and $p$, $X \sim \text{Binomial}(n,p)$, and its probability density function is given by

$$\mathbb{P}(X=k) = \binom{n}{k} p^k \left( 1-p \right)^{n-k}.$$

Such random variable has expected value

$$\mathbb{E}[X] = np,$$

and variance

$$\text{Var}[X]=np(1-p).$$
{{< /admonition >}}

{{< admonition type=note title="The Hypergeometric Distribution" open=true >}}
Let $X$ be a discrete random variable that represents the probability of obtaining $k$ successes in a series of $n$ draws without replacement, from a total population of $N$ samples, $K$ of which are synonymous with sucess. Then, $X$ is said to follow a Hypergeometric distribution with parameters $N$, $K$, and $n$, $X \sim \text{Hypergeometric}(N,K,n)$, and its probability density function is given by

$$\mathbb{P}(X = k) = \frac{\binom{K}{k} \binom{N-K}{n-k}}{\binom{N}{n}}$$

Such random variable has expected value

$$\mathbb{E}[X] = \frac{nK}{N},$$

and variance

$$\text{Var}[X]= \frac{nK}{N} \frac{N - K}{N} \frac{N-n}{N-1}.$$
{{< /admonition >}}

## Reservoir sampling

Motivation, basic idea

## Algorithm R

description, correctness, example implementation

## References

* Kai Lai Chung
* Article on Algorithm R
