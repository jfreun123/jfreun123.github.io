---
layout: post
title:  "Binomial Trees"
date:   2026-09-25 00:00:00 -0400
categories: math
math: true
---

*WORK IN PROGRESS*

*The following was written by me, but reviewed by Claude for grammar and math mistakes.*

Binomial trees are still foundational to option pricing (even with Black-Scholes being known).  They are extremely simple to understand and, with optimized programs, extremely fast to compute.  For this post, we will just go over the basics (and this will serve kind of as my notes from Hull).

Here, as is usual in coming up with prices, we first try to create a simple riskless portfolio -- that is, a portfolio whose price is stable regardless of certain price moves.  For this, let's just make a portfolio that contains some amount of stock ($\Delta$ shares) whose price is $S_0$ and a short position in a single option on the stock (or any other derivative that is a function of the underlying) whose price is $f$.  Let's have the following:

- the option lasts for a time $T$
- during that time, the underlying can either move up to $S_0 u$ or down to $S_0 d$, where $u > 1$ and $0 < d < 1$
- if the stock moves up then the option pays off $f_u$, and if it moves down it pays off $f_d$

// TODO:  add note in appendix on trinomial trees (the price of the option should nothing happen).

For a more intuitive picture:

![A one-step binomial tree: a root node with stock price S0 and option price f branches up to S0u with option price fu and down to S0d with option price fd](/assets/images/one-step-binomial-tree-hull-13-2.svg)

*Figure adapted from Options, Futures, and Other Derivatives.*[^hull]

Now, our simple portfolio has the following value at the end:

$$S_0 u \Delta - f_u \quad \text{if the underlying goes up}$$

$$S_0 d \Delta - f_d \quad \text{if the underlying goes down}$$

To make this riskless, we need to ensure that the value of going up or down is the same, that is:

$$S_0 u \Delta - f_u = S_0 d \Delta - f_d$$

or

$$\Delta = \frac{f_u - f_d}{S_0 u - S_0 d}$$

Now, we can make our single and only assumption:

- assume no arbitrage opportunities

This isn't realistic in the short term (microseconds) but is fine for longer-term pricing.

We need this assumption so that we can assume that the original price of the portfolio must grow at the risk-free rate.  That is, setting up the portfolio would cost

$$S_0 \Delta - f$$

And, because there are no arbitrage opportunities, this must eventually have value $(S_0 \Delta - f) e^{rT}$.  But we already know it should equal $S_0 u \Delta - f_u$.  So, setting these equal we have:

$$(S_0 \Delta - f) e^{rT} = S_0 u \Delta - f_u$$

or

$$S_0 \Delta - f = (S_0 u \Delta - f_u) e^{-rT}$$

or

$$f = S_0 \Delta (1 - u e^{-rT}) + f_u e^{-rT}$$

Substituting our expression for $\Delta$:

$$f = S_0 \left( \frac{f_u - f_d}{S_0 u - S_0 d} \right) (1 - u e^{-rT}) + f_u e^{-rT}$$

or

$$f = \frac{f_u (1 - d e^{-rT}) + f_d (u e^{-rT} - 1)}{u - d}$$

or

$$f = e^{-rT} \left[ p f_u + (1 - p) f_d \right]$$

where

$$p = \frac{e^{rT} - d}{u - d}$$

## Appendix: Sources

[^hull]: John C. Hull, *Options, Futures, and Other Derivatives*, Global Edition, 11th ed., Pearson, Chapter 13, "Binomial Trees".
