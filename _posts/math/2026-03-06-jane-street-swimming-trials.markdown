---
layout: post
title:  "Jane Street: Robot Updated Swimming Trials"
date:   2026-03-06 12:00:00 -0500
categories: math
math: true
---

Jane Street's May 2022 puzzle asks a deceptively clean question: in a robot swimming tournament governed by Nash equilibrium, what is the probability _p_ that a robot devotes all its fuel to a single race?

The answer is **p ≈ 0.999560**, meaning robots almost always go all-in on one race — but not quite.

---

## The Setup

24 robots compete in 8 races (N=8, so 3N robots and N races). Each robot has a fixed fuel budget and allocates it across the 8 races however it chooses. In each race, the robot that spends the *most* fuel wins and advances to the finals. Ties are broken uniformly at random.

A robot's goal is to maximize its probability of advancing — i.e., winning at least one race.

---

## What is Nash Equilibrium?

A **Nash equilibrium** is a set of strategies — one per player — where no individual player can improve their outcome by changing only their own strategy, assuming everyone else stays the same.

The classic example is rock-paper-scissors. If you always throw rock, your opponent can exploit that by always throwing paper. The only unexploitable strategy is to randomize uniformly — and that mixed strategy is the Nash equilibrium. Neither player can do better by deviating, because any pure choice gets beaten a third of the time.

The same logic applies here. If all robots played pure discrete (all fuel to one race), a clever robot could deviate: spread fuel thinly across all 8 races, guaranteed to show up everywhere, and win any race that no discrete robot happened to target. That's a profitable deviation, so pure discrete is *not* a Nash equilibrium.

The equilibrium must be a **mixed strategy**: play discrete with probability _p_ and continuous with probability $1 - p$. At this _p_, neither strategy is strictly better — a robot is completely indifferent between them. That indifference condition is what lets us solve for _p_.

---

## Two Strategies

Each robot can play one of two strategies:

- **Discrete**: Put all fuel into a single randomly-chosen race.
- **Continuous**: Spread fuel across multiple races.

At Nash equilibrium, no robot can improve its win probability by unilaterally switching strategies. This means every robot must be *indifferent* between discrete and continuous — both must yield the same win probability.

By symmetry, each of the 24 robots has an equal 1/3 chance of advancing (since 8 out of 24 advance). So the Nash equilibrium condition is:

> **The win probability under the continuous strategy = 1/3**

This is the key insight: we don't need to fully characterize the continuous strategy. We just need to compute the continuous win probability as a function of _p_, then solve for _p_.

---

## Computing the Continuous Win Probability

Suppose our robot plays continuous. The other 23 robots each independently play discrete with probability _p_ and continuous otherwise.

One more simplifying assumption: a robot playing continuous places nonzero weight on every race with probability 1. This means a continuous-strategy robot always "shows up" in all 8 races.

Now we compute W_C, the win probability for our continuous robot:

$$W_C = \sum_{i=0}^{23} \binom{23}{i} p^i (1-p)^{23-i} \sum_{j=0}^{7} \binom{8}{j} f(i,j) \left(\frac{1}{8}\right)^i \cdot \min\left(1, \frac{8-j}{24-i}\right)$$

Let's break this down:

- **Outer sum over _i_**: exactly _i_ of the 23 opponents play discrete. This happens with binomial probability.
- **Inner sum over _j_**: those _i_ discrete robots cover exactly _j_ distinct races (choosing which _j_ of 8 races get hit). The number of ways to assign _i_ distinct robots to exactly _j_ races with each race covered is $\binom{8}{j} f(i, j)$, and each robot picks a race uniformly from 8, giving the $(1/8)^i$ factor.
- **Win probability**: our continuous robot competes in all $8 - j$ uncovered races against only the $23 - i$ continuous opponents. The $\min(1, \cdot)$ term caps the probability at 1 — if there are fewer continuous opponents than remaining races, we're guaranteed to win at least one.

---

## The f(i, j) Function

$f(i, j)$ is the number of **surjections** from _i_ distinct objects onto _j_ distinct categories — i.e., ways to assign _i_ robots to _j_ races so every race gets at least one robot.

It satisfies the recurrence:

$$
f(0, 0) = 1, \quad f(n, 0) = f(0, n) = 0 \text{ for } n > 0
$$
$$
f(i, j) = j \cdot (f(i-1, j) + f(i-1, j-1)) \quad \text{for } i, j > 0
$$

This is just the Stirling numbers of the second kind scaled by $j!$.

---

## Solving for p

Setting $W_C = 1/3$ and solving numerically:

$$
p \approx 0.999560
$$

So at Nash equilibrium, robots play the pure discrete strategy with about 99.96% probability. The continuous strategy gets a tiny but nonzero weight — just enough to make everyone indifferent.

This makes intuitive sense: in a race where everyone else is going all-in on a single random race, spreading your fuel thin is almost never the right move. But "almost never" is not "never."

---

## Resources

- [Jane Street Puzzle Statement](https://www.janestreet.com/puzzles/robot-updated-swimming-trials-index/)
- [Official Solution](https://www.janestreet.com/puzzles/robot-updated-swimming-trials-solution/)
