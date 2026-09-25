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

- two riskless portfolios (that is, two portfolios with a guaranteed final value) that cost the same amount to set up must end with the same final value (this is the no-arbitrage assumption).

After all, if it were possible to violate this, then you could guarantee free, riskless money.  Such opportunities do exist in high-frequency trading (HFT), but they are incredibly difficult to capture and can be assumed not to exist for the purpose of this calculation.

So, creating this portfolio requires this much cash up front:

$$S_0 \Delta - f$$

With that much cash, instead of creating this underlying-and-option portfolio, I could just invest it at the risk-free rate $r$ and guarantee a final value of $(S_0 \Delta - f) e^{rT}$.  But we already know this should equal $S_0 u \Delta - f_u$, because both portfolios are riskless and cost the same amount to set up:

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

Notice how the equation for the option price, $f$, does *not* depend on the probability of the underlying swinging up or down.  Instead, it just depends on the magnitude of those swings.  This is where the "vol-centric" trading approach comes from:  it does not matter where the underlying is going to go, it just matters how large those jumps could be.  That is "all" that is important, and we already see this in this simple binomial model!  While this may seem counterintuitive, the standard argument is that the underlying stock price itself already incorporates the probabilities of these future movements (and therefore, we would be double counting if we incorporated them into our option price).

Moreover, notice that our equation for the current price of the option, $f$, has a component that looks like an expectation:  $p f_u + (1 - p) f_d$.  This is no coincidence.  It is heavily related to the principle of risk-neutral valuation.  Risk-neutral valuation says that, when valuing a derivative, we can make the assumption that investors are risk-neutral and do not expect increased returns when they take on increased risk.  This is a weird concept because that is clearly not the world we live in, and yet pricing can be done in that world and still be true in this world (and all worlds)!

The two most important features of this risk-neutral world are:

1.  The expected return on a stock is just the risk-free rate:  $E(S_T) = S_0 e^{\mu T} = S_0 e^{rT}$, where $\mu$ is the expected return on the stock and $r$ is the risk-free rate.
2.  The discount rate used for the expected payoff of an option is the risk-free rate:  a payoff of \\$10 in two months is worth $\\$10 \, e^{-r \cdot 2/12}$ today (with $r = 12\%$ per year, that is $\\$10 \, e^{-0.02} \approx \\$9.80$).  The risk it took to get that payoff does not matter.

Now, we can reinterpret 

$$f = e^{-rT} \left[ p f_u + (1 - p) f_d \right]$$

with $p$ being the probability of an up movement in the underlying in a risk-neutral world.  Then $p f_u + (1 - p) f_d$ is just the expected payoff from the option in a risk-neutral world.  Similarly, the e^rT in the front of the equation is just the second feature being applied:  the value of the option today can just be discounted at the risk free rate.

We can check that this makes sense by calculating the expected stock price at time $T$, $E(S_T)$:

$$E(S_T) = p S_0 u + (1 - p) S_0 d = p S_0 (u - d) + S_0 d$$

Substituting our original expression for $p$:

$$E(S_T) = \frac{e^{rT} - d}{u - d} \, S_0 (u - d) + S_0 d = S_0 e^{rT}$$

which recovers our first point exactly:  the expected return of the stock is just the risk-free rate.

## Appendix: Sources

[^hull]: John C. Hull, *Options, Futures, and Other Derivatives*, Global Edition, 11th ed., Pearson, Chapter 13, "Binomial Trees".
