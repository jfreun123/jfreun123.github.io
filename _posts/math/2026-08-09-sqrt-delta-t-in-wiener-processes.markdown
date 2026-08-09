---
layout: post
title:  "Where the sqrt(Δt) in a Wiener Process Comes From"
date:   2026-08-09 12:00:00 -0500
categories: math
math: true
---

*The following was written by me, but reviewed by Claude for grammar and math mistakes.*

While pure financial modeling may have limited edge in today's world (with classical machine learning being a dominant source of edge to feed into classical financial models), it is still important -- or at the very least, interesting -- to know where these classical models came from.  To start, there are a few prerequisite definitions.  First off, it is important to note what a Markov process is-- it is just a process where only the current value of a variable is relevant for predicting the future value.  This assumption is generally used as a reflection of the weak form of market efficiency, which states that the present price of stocks contains all the information contained in a record of past prices.  Evidently, this claim is very likely not true based on various papers[^lomackinlay][^itosugiyama] and the existence of billion-dollar industries.  Nonetheless, it is the first foundational assumption that all of the following work is built on.  

Next, for another useful property:  if the changes over two non-overlapping periods are independent and normally distributed, $X_1 \sim \mathcal{N}(\mu_1, \sigma_1^2)$ and $X_2 \sim \mathcal{N}(\mu_2, \sigma_2^2)$, then the total change over the combined period is distributed as

$$X_1 + X_2 \sim \mathcal{N}\left(\mu_1 + \mu_2,\ \sigma_1^2 + \sigma_2^2\right)$$

That is, the [means add](#linearity-of-expectation) and the [variances add](#variances-add).

## The Wiener process

Now, to get into the weeds of this article, we define a Wiener process as a particular type of Markov process with a mean change of zero and a variance rate of 1.0.  Formally, it has the following properties:

**Property 1:**  The change $\Delta z$ during a small period of time $\Delta t$ is

$$\Delta z = \epsilon \sqrt{\Delta t}$$

where $\epsilon$ is a draw from a standardized normal distribution, $\epsilon \sim \mathcal{N}(0, 1)$.

**Property 2:**  The values of $\Delta z$ for any two different short intervals of time, $\Delta t$, are independent.

As an immediate consequence of Property 1, we have:

- $E[\Delta z] = E\left[\epsilon \sqrt{\Delta t}\right] = \sqrt{\Delta t}\, E[\epsilon] = \sqrt{\Delta t} \cdot 0 = 0$, since $\sqrt{\Delta t}$ is a constant (so it factors out of the expectation by [linearity](#linearity-of-expectation)) and $\epsilon \sim \mathcal{N}(0, 1)$ has [mean zero](#mean-of-standard-normal)
- $\text{Var}(\Delta z) = E\left[(\Delta z - E[\Delta z])^2\right] = E\left[\Delta z^2\right] = E\left[\epsilon^2 \Delta t\right] = \Delta t\, E\left[\epsilon^2\right] = \Delta t$
- $\text{SD}(\Delta z) = \sqrt{\text{Var}(\Delta z)} = \sqrt{\Delta t}$

where the second line starts from the [definition of variance](#definition-of-variance) and uses the fact that $\Delta t$ is a constant along with the [second moment of the standard normal](#second-moment), $E\left[\epsilon^2\right] = \text{Var}(\epsilon) + E[\epsilon]^2 = 1 + 0 = 1$.

Moreover, Property 2 immediately tells us that a Wiener process is Markov.

(For a cool related fact -- the expected *length* of the path followed by $z$ in any time interval is infinite -- see the [appendix](#appendix-the-expected-path-length-of-a-wiener-process-is-infinite).)

## Generalizing it

Now, to generalize Wiener for a variable $x$:

$$dx = a\,dt + b\,dz$$

![A generalized Wiener process with a = 0.3 and b = 1.5: the straight drift line dx = a dt, a Wiener process dz wandering below it, and the generalized process dx = a dt + b dz combining both](/assets/images/generalized-wiener-hull-13-2.svg)

*Figure adapted from Options, Futures, and Other Derivatives.*[^hull]

Here, $b\,dz$ is considered the "noise" term.  If it were zero, we'd have a very well defined $dx = a\,dt$ and could immediately solve to get $x = x_0 + at$.  Moreover, a Wiener process has a variance rate per unit time of 1.0.  It follows, since [variance scales quadratically](#variance-scaling), that "$b$ times a Wiener process" has a variance rate per unit time of $b^2$ (remember, we interpret models in terms of standard deviation but we use variance because it is algebraically simpler to work with).  So, in a generalized Wiener process, Property 1 becomes

$$\Delta x = a\,\Delta t + b \epsilon \sqrt{\Delta t}$$

Repeating the same derivation as before:

- $E[\Delta x] = a\,\Delta t + b \sqrt{\Delta t}\, E[\epsilon] = a\,\Delta t$
- $\text{Var}(\Delta x) = E\left[(\Delta x - E[\Delta x])^2\right] = E\left[b^2 \epsilon^2 \Delta t\right] = b^2\, \Delta t\, E\left[\epsilon^2\right] = b^2\, \Delta t$
- $\text{SD}(\Delta x) = b \sqrt{\Delta t}$

where the variance line also uses that [deterministic shifts don't move variance](#deterministic-shifts): the drift term $a\,\Delta t$ is a constant, so only the noise term survives $\Delta x - E[\Delta x]$.

Moreover, over a general $T$ (not a small $\Delta t$), we have

$$x(T) - x(0) \sim \mathcal{N}\left(aT,\ b^2 T\right)$$

To see this, split $[0, T]$ into $N = T / \Delta t$ small intervals.  By Property 2, the increments $\Delta x_i$ are independent, so by the additivity property from earlier, the [means](#linearity-of-expectation) and the [variances](#variances-add) of the $N$ increments add:

- $E[x(T) - x(0)] = \sum_{i=1}^{N} E[\Delta x_i] = N \cdot a\,\Delta t = aT$
- $\text{Var}(x(T) - x(0)) = \sum_{i=1}^{N} \text{Var}(\Delta x_i) = N \cdot b^2\,\Delta t = b^2 T$

and a [sum of independent normal variables is itself normal](#normal-sums).  The standard deviation of $x(T) - x(0)$ is therefore $b \sqrt{T}$: uncertainty grows with the *square root* of elapsed time.

## Why sqrt(Δt)?

Now, for the goal of this post:  why $\sqrt{\Delta t}$?  Why not just have

$$\Delta x = a\,\Delta t + b \epsilon\, \Delta t$$

Let's explore the consequences of that:

If we did that, we'd now have a per-step variance of

$$\text{Var}(\Delta x) = E\left[(\Delta x - E[\Delta x])^2\right] = E\left[b^2 \epsilon^2 \Delta t^2\right] = b^2\, \Delta t^2\, E\left[\epsilon^2\right] = b^2\, \Delta t^2$$

by exactly the same steps as before.

This would lead to an overall variance over a period $T$ (again splitting it into $N = T / \Delta t$ independent increments whose [variances add](#variances-add)) of

$$\text{Var}(x(T) - x(0)) = N \cdot b^2\, \Delta t^2 = \frac{T}{\Delta t} \cdot b^2\, \Delta t^2 = b^2\, T\, \Delta t$$

which would go to zero as $\Delta t$ goes to zero!  That is to say, we would be able to predict exactly the terminal value of this process and nothing is very interesting-- we would just have a straight line.

Ok, but what if we do some other power?  What about the fourth root of $\Delta t$?  In general, let's take a look at

$$\Delta x = a\,\Delta t + b \epsilon\, (\Delta t)^p$$

where $p \in \mathbb{R}^+$.  This would give us a per-step variance of

$$\text{Var}(\Delta x) = E\left[b^2 \epsilon^2 (\Delta t)^{2p}\right] = b^2\, (\Delta t)^{2p}\, E\left[\epsilon^2\right] = b^2\, (\Delta t)^{2p}$$

which would give us a total variance over a period $T$ of

$$\text{Var}(x(T) - x(0)) = N \cdot b^2\, (\Delta t)^{2p} = \frac{T}{\Delta t} \cdot b^2\, (\Delta t)^{2p} = b^2\, T\, (\Delta t)^{2p - 1}$$

Now everything hinges on the exponent $2p - 1$:

- If $p > 1/2$, then $(\Delta t)^{2p - 1} \to 0$ as $\Delta t \to 0$:  the variance vanishes and we are back to a deterministic straight line.
- If $p < 1/2$, then $(\Delta t)^{2p - 1} \to \infty$ as $\Delta t \to 0$:  the variance blows up over any finite interval.
- Only $p = 1/2$ makes the exponent exactly zero, leaving $\text{Var}(x(T) - x(0)) = b^2\, T$:  finite, nonzero, and independent of how finely we slice the interval.

That is, the only reasonable, interesting value here is $p = 1/2$.  Hence why the square root makes its way in.  We could probably use non-polynomial functions but that would just complicate future work.  $\sqrt{\Delta t}$ is the simplest we can get here.

---

## Appendix: Simulate it yourself

The whole construction is just Property 1 applied $N$ times and accumulated with a cumulative sum -- a few lines of NumPy.  Seed 14 reproduces the exact paths in the figure above; change the seed, $a$, or $b$ and see what happens:

```python
import numpy as np
import matplotlib.pyplot as plt

a, b = 0.3, 1.5
T, N = 25.0, 1500
dt = T / N

t = np.linspace(0, T, N + 1)
eps = np.random.default_rng(seed=14).standard_normal(N)

z = np.concatenate([[0.0], np.cumsum(eps * np.sqrt(dt))])
x = a * t + b * z

plt.plot(t, a * t, label=r"$dx = a\,dt$ (drift only)")
plt.plot(t, x, label=r"$dx = a\,dt + b\,dz$ (generalized Wiener)")
plt.plot(t, z, label=r"Wiener process, $z$")
plt.xlabel("Time")
plt.ylabel(r"Value of variable, $x$")
plt.legend()
plt.show()
```

To see the point of this post in action, replace `np.sqrt(dt)` with `dt ** p`:  for $p > 1/2$ the noise flattens out and the path hugs the drift line, and for $p < 1/2$ it swamps everything.  Crank $N$ up while you're at it -- only $p = 1/2$ keeps the picture stable as the steps get finer.

Or play with it right here, no Python required.  The random shocks $\epsilon_i$ are held fixed while you move the slider, so what you are watching is purely the effect of $p$; hit the button for a fresh set of shocks.  The readout under the chart tracks the total variance $b^2\, T\, (\Delta t)^{2p - 1}$ from the derivation above:

<div id="ww" style="border: 1px solid #ddd; border-radius: 6px; padding: 14px; margin: 1em 0; background: #ffffff; font-size: 14px; color: #444444;">
  <div style="display: flex; flex-wrap: wrap; gap: 16px; align-items: center; margin-bottom: 10px;">
    <label>p = <output id="ww-pv">0.50</output>
      <input id="ww-p" type="range" min="0.05" max="1" step="0.05" value="0.5" style="vertical-align: middle; width: 160px;">
    </label>
    <label>steps N:
      <select id="ww-n">
        <option value="150">150</option>
        <option value="1500" selected>1500</option>
        <option value="15000">15000</option>
      </select>
    </label>
    <button id="ww-roll" type="button">New random shocks</button>
  </div>
  <div style="margin-bottom: 8px;">
    <span style="display: inline-block; width: 18px; height: 3px; background: #111111; vertical-align: middle;"></span> drift only (&Delta;x = a&Delta;t)
    &nbsp;&nbsp;
    <span style="display: inline-block; width: 18px; height: 3px; background: #2563eb; vertical-align: middle;"></span> with noise (&Delta;x = a&Delta;t + b&epsilon;(&Delta;t)<sup>p</sup>)
  </div>
  <canvas id="ww-c" width="720" height="400" role="img" aria-label="Simulated generalized Wiener process path against its drift line" style="width: 100%; max-width: 720px; height: auto;"></canvas>
  <div id="ww-info" style="margin-top: 6px;"></div>
  <div id="ww-hover" style="color: #888888; min-height: 1.2em;"></div>
</div>

<script>
(function () {
  "use strict";
  var A = 0.3, B = 1.5, T = 25;
  var W = 720, H = 400, ML = 46, MR = 12, MT = 10, MB = 26;
  var canvas = document.getElementById("ww-c");
  var ctx = canvas.getContext("2d");
  var pEl = document.getElementById("ww-p");
  var pvEl = document.getElementById("ww-pv");
  var nEl = document.getElementById("ww-n");
  var rollEl = document.getElementById("ww-roll");
  var infoEl = document.getElementById("ww-info");
  var hoverEl = document.getElementById("ww-hover");
  var N = parseInt(nEl.value, 10);
  var eps = [];
  var hoverIdx = null;
  function randn() {
    var u = 0, v = 0;
    while (u === 0) u = Math.random();
    while (v === 0) v = Math.random();
    return Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v);
  }
  function reroll() {
    eps = [];
    for (var i = 0; i < N; i++) eps.push(randn());
  }
  function buildPath(p) {
    var dt = T / N, step = Math.pow(dt, p), x = [0];
    for (var i = 0; i < N; i++) x.push(x[i] + A * dt + B * eps[i] * step);
    return x;
  }
  function niceStep(raw) {
    var mag = Math.pow(10, Math.floor(Math.log(raw) / Math.LN10));
    var r = raw / mag;
    if (r < 1.5) return mag;
    if (r < 3.5) return 2 * mag;
    if (r < 7.5) return 5 * mag;
    return 10 * mag;
  }
  function formatNum(v) {
    if (v === 0) return "0";
    var av = Math.abs(v);
    if (av >= 1000 || av < 0.01) return v.toExponential(1);
    if (av >= 10) return v.toFixed(1);
    return v.toFixed(2);
  }
  function draw() {
    var p = parseFloat(pEl.value);
    var dt = T / N;
    var x = buildPath(p);
    var yMin = 0, yMax = A * T, i, v;
    for (i = 0; i <= N; i++) {
      if (x[i] < yMin) yMin = x[i];
      if (x[i] > yMax) yMax = x[i];
    }
    var pad = (yMax - yMin) * 0.06 || 1;
    yMin -= pad;
    yMax += pad;
    var dpr = window.devicePixelRatio || 1;
    canvas.width = W * dpr;
    canvas.height = H * dpr;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    ctx.clearRect(0, 0, W, H);
    function X(t) { return ML + (t / T) * (W - ML - MR); }
    function Y(val) { return MT + ((yMax - val) / (yMax - yMin)) * (H - MT - MB); }
    ctx.strokeStyle = "#cccccc";
    ctx.lineWidth = 1;
    ctx.fillStyle = "#888888";
    ctx.font = "12px sans-serif";
    ctx.beginPath();
    ctx.moveTo(ML, MT);
    ctx.lineTo(ML, H - MB);
    ctx.lineTo(W - MR, H - MB);
    ctx.stroke();
    ctx.textAlign = "center";
    ctx.textBaseline = "top";
    for (i = 0; i <= 25; i += 5) ctx.fillText(String(i), X(i), H - MB + 6);
    var stepY = niceStep((yMax - yMin) / 4);
    ctx.textAlign = "right";
    ctx.textBaseline = "middle";
    for (v = Math.ceil(yMin / stepY) * stepY; v <= yMax; v += stepY) {
      ctx.fillText(formatNum(Math.abs(v) < stepY / 1e6 ? 0 : v), ML - 6, Y(v));
      ctx.beginPath();
      ctx.moveTo(ML - 3, Y(v));
      ctx.lineTo(ML, Y(v));
      ctx.stroke();
    }
    if (yMin < 0 && yMax > 0) {
      ctx.strokeStyle = "#e3e3e3";
      ctx.beginPath();
      ctx.moveTo(ML, Y(0));
      ctx.lineTo(W - MR, Y(0));
      ctx.stroke();
    }
    ctx.strokeStyle = "#111111";
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(X(0), Y(0));
    ctx.lineTo(X(T), Y(A * T));
    ctx.stroke();
    ctx.strokeStyle = "#2563eb";
    ctx.lineWidth = 1.5;
    ctx.beginPath();
    for (i = 0; i <= N; i++) {
      if (i === 0) ctx.moveTo(X(i * dt), Y(x[i]));
      else ctx.lineTo(X(i * dt), Y(x[i]));
    }
    ctx.stroke();
    if (hoverIdx !== null) {
      ctx.fillStyle = "#2563eb";
      ctx.beginPath();
      ctx.arc(X(hoverIdx * dt), Y(x[hoverIdx]), 4, 0, 2 * Math.PI);
      ctx.fill();
      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 2;
      ctx.stroke();
      hoverEl.textContent = "t = " + (hoverIdx * dt).toFixed(2) + ",  x = " + x[hoverIdx].toFixed(3);
    } else {
      hoverEl.textContent = "";
    }
    pvEl.textContent = p.toFixed(2);
    var variance = B * B * T * Math.pow(dt, 2 * p - 1);
    infoEl.textContent = "Total variance over the period: b² · T · Δt^(2p−1) ≈ " + formatNum(variance) + "  (SD ≈ " + formatNum(Math.sqrt(variance)) + ")";
  }
  canvas.addEventListener("mousemove", function (e) {
    var rect = canvas.getBoundingClientRect();
    var lx = (e.clientX - rect.left) * (W / rect.width);
    var idx = Math.round(((lx - ML) / (W - ML - MR)) * N);
    hoverIdx = (idx >= 0 && idx <= N) ? idx : null;
    draw();
  });
  canvas.addEventListener("mouseleave", function () { hoverIdx = null; draw(); });
  pEl.addEventListener("input", draw);
  nEl.addEventListener("change", function () { N = parseInt(nEl.value, 10); reroll(); draw(); });
  rollEl.addEventListener("click", function () { reroll(); draw(); });
  reroll();
  draw();
})();
</script>

## Appendix: The Probability Toolbox

For reference, here is every statistics and probability fact the derivations above leaned on, and where each one did its work:

- **Linearity of expectation:**{: #linearity-of-expectation} $E[X + Y] = E[X] + E[Y]$ and $E[cX] = c\,E[X]$ for any constant $c$ -- no independence assumption required.  Used to pull $\sqrt{\Delta t}$ out of $E[\Delta z]$, to split $E[\Delta x]$ into its drift and noise pieces, and to turn $E[x(T) - x(0)]$ into a sum of the $N$ per-interval means.

- **Mean of the standard normal:**{: #mean-of-standard-normal} $E[\epsilon] = 0$ for $\epsilon \sim \mathcal{N}(0, 1)$.  This is what kills the noise term in both $E[\Delta z] = 0$ and $E[\Delta x] = a\,\Delta t$.

- **Definition of variance:**{: #definition-of-variance} $\text{Var}(X) = E\left[(X - E[X])^2\right]$.  The starting point of both variance computations.

- **Second moment of the standard normal:**{: #second-moment} $E[\epsilon^2] = \text{Var}(\epsilon) + E[\epsilon]^2 = 1$.  This is the shortcut formula $\text{Var}(X) = E[X^2] - E[X]^2$ run in reverse; it converts $E[\epsilon^2 \Delta t]$ into $\Delta t$ in both variance bullets.

- **Deterministic shifts don't move variance:**{: #deterministic-shifts} $\text{Var}(X + c) = \text{Var}(X)$.  Used implicitly in $\text{Var}(\Delta x)$: the drift term $a\,\Delta t$ is deterministic, so only the noise term $b \epsilon \sqrt{\Delta t}$ contributes.

- **Scaling is linear in expectation but quadratic in variance:**{: #variance-scaling} $\text{Var}(cX) = c^2\,\text{Var}(X)$.  This is why "$b$ times a Wiener process" has a variance rate of $b^2$, and -- taking the square root back out -- why every standard deviation in this post ends up under a radical.

- **Variances add under independence:**{: #variances-add} $\text{Var}(X + Y) = \text{Var}(X) + \text{Var}(Y) + 2\,\text{Cov}(X, Y)$, and the covariance term vanishes for independent variables.  This is the one place independence (Property 2) is truly load-bearing: it lets the $N$ per-interval variances sum to $b^2 T$ in the $x(T) - x(0)$ proof.  Contrast with expectations, which add unconditionally.

- **Sums of independent normals are normal:**{: #normal-sums} the normal family is closed under independent addition.  This upgrades "mean $aT$, variance $b^2 T$" to the full distributional statement $x(T) - x(0) \sim \mathcal{N}\left(aT,\ b^2 T\right)$.

- **Mean absolute value of the standard normal:**{: #half-normal-mean} $E\lvert\epsilon\rvert = \sqrt{2/\pi} \approx 0.798$ -- the mean of the half-normal distribution.  Without doing the full work, here is the integral it comes from:

  $$E\left|\epsilon\right| = \int_{-\infty}^{\infty} \lvert x \rvert\, \frac{e^{-x^2/2}}{\sqrt{2\pi}}\, dx = \frac{2}{\sqrt{2\pi}} \int_{0}^{\infty} x\, e^{-x^2/2}\, dx = \frac{2}{\sqrt{2\pi}} \cdot 1 = \sqrt{\frac{2}{\pi}}$$

  The density is symmetric, so integrating $\lvert x \rvert$ is twice the positive half; then the substitution $u = x^2/2$ turns the remaining integral into $\int_0^\infty e^{-u}\, du = 1$.  (The $\pi$ is hiding in the density's normalizing constant $\sqrt{2\pi}$ -- *that* one is the famous change-of-coordinates trick, where you square the Gaussian integral and switch to polar coordinates.)  Used in the path-length appendix to compute the expected size of a single step, $E\lvert\Delta z\rvert$.

## Appendix: The expected path length of a Wiener process is infinite

This is a cool fact that I could not resist writing about, but that I also don't want to give a full blog post.  Going back to a simple Wiener (not generalized):

$$\Delta z = \epsilon \sqrt{\Delta t}$$

What is the sum of this path over any period $T$?

The distance the path travels is the sum of the absolute values of its steps.  Splitting $[0, T]$ into $N = T / \Delta t$ steps as usual, the path length is

$$L = \sum_{i=1}^{N} \left|\Delta z_i\right| = \sum_{i=1}^{N} \left|\epsilon_i\right| \sqrt{\Delta t}$$

Taking expectations, by [linearity](#linearity-of-expectation) and the [mean absolute value of the standard normal](#half-normal-mean), $E\lvert\epsilon\rvert = \sqrt{2/\pi}$:

$$E[L] = N \cdot \sqrt{\frac{2}{\pi}}\, \sqrt{\Delta t} = \frac{T}{\Delta t} \cdot \sqrt{\frac{2}{\pi}}\, \sqrt{\Delta t} = \sqrt{\frac{2}{\pi}}\, \frac{T}{\sqrt{\Delta t}}$$

which goes to infinity as $\Delta t \to 0$.  So over any interval, no matter how short, the expected distance traveled by a Wiener process is infinite -- even though it only ends up on the order of $\sqrt{T}$ away from where it started.  (In the language of analysis: Wiener paths have unbounded variation.)

Does this survive the generalization $\Delta x = a\,\Delta t + b \epsilon \sqrt{\Delta t}$?  Yes, whenever $b \neq 0$.  Per step, the drift contributes order $\Delta t$ while the noise contributes order $\sqrt{\Delta t}$, so at small scales the noise dominates every step.  Using the reverse triangle inequality, $\lvert X + c \rvert \geq \lvert X \rvert - \lvert c \rvert$:

$$E[L] = \sum_{i=1}^{N} E\left|a\,\Delta t + b \epsilon_i \sqrt{\Delta t}\right| \geq \frac{T}{\Delta t} \left( \lvert b \rvert \sqrt{\frac{2}{\pi}}\, \sqrt{\Delta t} - \lvert a \rvert\, \Delta t \right) = \sqrt{\frac{2}{\pi}}\, \frac{\lvert b \rvert\, T}{\sqrt{\Delta t}} - \lvert a \rvert\, T$$

which still goes to infinity as $\Delta t \to 0$.  The drift can only ever contribute $\lvert a \rvert\, T$ of length in total -- a straight line's worth -- while the noise contributes on the order of $T / \sqrt{\Delta t}$.  Only the degenerate case $b = 0$ has a finite expected path length.

## Appendix: Sources

[^hull]: John C. Hull, *Options, Futures, and Other Derivatives*, Global Edition, 11th ed., Pearson, Figure 13.2, page 321.

[^lomackinlay]: Andrew W. Lo and A. Craig MacKinlay, ["Stock Market Prices Do Not Follow Random Walks: Evidence from a Simple Specification Test"](https://academic.oup.com/rfs/article-abstract/1/1/41/1601244), The Review of Financial Studies, Vol. 1, No. 1, 1988, pp. 41-66.

[^itosugiyama]: Mikio Ito and Shunsuke Sugiyama, ["Measuring the degree of time varying market inefficiency"](https://www.sciencedirect.com/science/article/abs/pii/S0165176509000408), Economics Letters, Vol. 103, No. 1, 2009, pp. 62-64.
