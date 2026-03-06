---
layout: post
title:  "Jane Street: Robot Updated Swimming Trials"
date:   2026-03-06 12:00:00 -0500
categories: math
math: true
---

Jane Street's May 2022 puzzle asks: in a robot swimming tournament played at Nash equilibrium, what is the probability $p$ that a robot devotes all its fuel to a single race?

The answer is **p ≈ 0.999560** — robots almost always go all-in on one race, but not quite.

---

## The Setup

24 robots compete in 8 races. Each robot has a fixed fuel budget and allocates it across the races however it chooses. In each race, the robot that spends the *most* fuel wins and advances to the finals. Ties are broken uniformly at random.

8 out of 24 robots advance, so each robot's baseline win probability is $1/3$.

---

## Nash Equilibrium and the Key Insight

A **Nash equilibrium** is a set of strategies where no player can improve their outcome by unilaterally changing their own strategy. In a symmetric game like this, the equilibrium is the same for every robot.

**Why pure discrete fails:** If every robot put all its fuel into one randomly-chosen race, a clever robot could deviate: spread fuel thinly across all 8 races, show up in every race, and win any race that nobody else targeted. That's a profitable deviation — so pure discrete is *not* an equilibrium.

**The mixed strategy:** The equilibrium must mix between two strategies:

- **Discrete**: put all fuel into a single randomly-chosen race.
- **Continuous**: spread fuel across multiple races.

Each robot plays discrete with probability $p$ and continuous with probability $1 - p$. At the equilibrium $p$, a robot is completely *indifferent* between the two strategies — both yield the same win probability.

Since 8 of 24 robots advance, that win probability is exactly $1/3$. So the Nash equilibrium condition is simply:

$$W_C(p) = \frac{1}{3}$$

where $W_C(p)$ is the win probability of a continuous robot when all opponents play the mixed strategy. We don't need to characterize the continuous strategy in detail — we just need to compute $W_C(p)$ and solve for $p$.

---

## The f(i, j) Function

Before deriving $W_C$, we need a counting tool.

$f(i, j)$ is the number of ways to assign $i$ labeled robots to $j$ labeled races such that every race gets at least one robot — a **surjection** (onto function).

**Example — $f(2, 2)$:** Two robots (A, B) assigned to two races (1, 2), every race covered:

| Robot A | Robot B | Race 1 covered? | Race 2 covered? |
|---------|---------|-----------------|-----------------|
| Race 1  | Race 2  | ✓ | ✓ |
| Race 2  | Race 1  | ✓ | ✓ |
| Race 1  | Race 1  | ✓ | ✗ |
| Race 2  | Race 2  | ✗ | ✓ |

Only the first two are valid, so $f(2, 2) = 2$.

**More values:**
- $f(1, 1) = 1$ — one robot, one race, trivially covered.
- $f(2, 1) = 1$ — both robots go to the only race.
- $f(3, 2) = 6$ — equals $2^3 - 2$ (all assignments minus those where everyone picks the same race).
- $f(3, 3) = 6$ — equals $3!$ (one robot per race, in any order).

**Recurrence:** For each new robot $i$, it either joins a race already covered ($j$ choices, leaving $f(i-1,j)$ ways for the rest) or opens a brand-new race ($j$ choices for which race it opens, leaving $f(i-1,j-1)$ ways for the rest):

$$f(0,0) = 1, \quad f(n,0) = f(0,n) = 0 \text{ for } n > 0$$

$$f(i,j) = j \cdot \bigl(f(i-1,j) + f(i-1,j-1)\bigr) \quad \text{for } i,j > 0$$

This is the Stirling numbers of the second kind scaled by $j!$: $f(i,j) = j!\cdot S(i,j)$.

---

## Computing $W_C(p)$

Our robot plays continuous. The 23 opponents each independently play discrete with probability $p$. We assume a continuous robot always places nonzero fuel in every race (an assumption the official solution takes as given).

The only thing that matters about the discrete robots is *which races they cover* — they outspend us in those races, so our only opportunities are the uncovered ones. We condition on:

- $i$: how many opponents play discrete
- $j$: how many distinct races those $i$ robots collectively cover

$$W_C = \sum_{i=0}^{23} \left[ \underbrace{\binom{23}{i} p^i (1-p)^{23-i}}_{\text{prob. of } i \text{ discrete opponents}} \cdot \sum_{j=0}^{7} \underbrace{\binom{8}{j} f(i,j) \left(\frac{1}{8}\right)^i}_{\text{prob. they cover } j \text{ races}} \cdot \underbrace{\min\!\left(1,\, \frac{8-j}{24-i}\right)}_{\text{our win prob.}} \right]$$

**Outer sum — probability of exactly $i$ discrete opponents:**

Each of the 23 opponents independently plays discrete with probability $p$, so the count is binomial:

$$\binom{23}{i} p^i (1-p)^{23-i}$$

**Inner sum — probability that those $i$ robots cover exactly $j$ distinct races:**

Each discrete robot picks a race uniformly from 8. We want the probability they collectively land on exactly $j$ distinct races. To count this:

1. Choose *which* $j$ races get covered: $\binom{8}{j}$ ways.
2. Assign $i$ robots to those $j$ races so every chosen race gets at least one robot: $f(i,j)$ ways (the surjection count from the previous section).
3. Each robot picks its race independently and uniformly: probability $(1/8)^i$.

So the probability that $i$ discrete robots cover exactly $j$ specific races is $\binom{8}{j} \cdot f(i,j) \cdot (1/8)^i$.

**Win probability given $(i, j)$:**

The $j$ covered races are locked — discrete robots outspend us there. We only have a chance in the $8 - j$ *uncovered* races, competed for by all $24 - i$ continuous robots (us plus $23 - i$ others). By symmetry among continuous robots, our expected win probability is:

$$\frac{8-j}{24-i}$$

The $\min(1, \cdot)$ caps this at 1 for edge cases where continuous robots are outnumbered by uncovered races, guaranteeing we win at least one.

**Worked example — $i=1$, $j=1$:**

One opponent plays discrete and claims one race. We and $22$ other continuous robots compete for the remaining $7$ uncovered races. Our win probability is $7/23 \approx 0.304 < 1/3$ — the discrete robot consumed a race without us getting credit for it.

---

## Solving for p

Setting $W_C(p) = 1/3$ numerically gives:

$$\boxed{p \approx 0.999560}$$

At Nash equilibrium, robots play discrete with ~99.96% probability. The continuous strategy gets a tiny but nonzero weight — just enough to make everyone indifferent.

**Why is $p$ so close to 1?** When $p$ is large, almost all opponents are discrete. With 23 robots each independently picking from 8 races, they cover on average $8\left(1-(7/8)^{23}\right) \approx 7.6$ of 8 races — leaving roughly 0.4 uncovered on average. A lone continuous robot wins those free races, giving $W_C > 1/3$. Equilibrium requires just a hair of continuous mixing to bring this back down to $1/3$.

---

## Implementation

> **Note:** This site is fully static — there is no server. The solver below runs entirely in your browser as JavaScript.

### Step 1 — Precompute the surjection table

Storing $f(i,j)$ directly overflows: $f(23,8) > 10^{20}$, which exceeds 64-bit float precision. Instead define $g(i,j) = f(i,j) \cdot (1/8)^i$. Dividing the recurrence through by $8^i$:

$$g(i,j) = \frac{j}{8}\left(g(i-1,j) + g(i-1,j-1)\right)$$

All values of $g$ stay in $[0,1]$, and the $(1/8)^i$ factor drops out of the formula.

```javascript
const G = Array.from({length: 24}, () => new Float64Array(9));
G[0][0] = 1;
for (let i = 1; i <= 23; i++) {
  for (let j = 1; j <= Math.min(i, 8); j++) {
    G[i][j] = (j / 8) * (G[i - 1][j] + G[i - 1][j - 1]);
  }
}
```

### Step 2 — Evaluate $W_C(p)$

Direct translation of the formula above, using `G[i][j]` in place of `f(i,j) * (1/8)^i`.

```javascript
function wc(p) {
  let total = 0;
  for (let i = 0; i <= 23; i++) {
    const pI = binom(23, i) * Math.pow(p, i) * Math.pow(1 - p, 23 - i);
    if (pI < 1e-300) continue;
    let inner = 0;
    for (let j = 0; j <= Math.min(i, 7); j++) {
      inner += binom(8, j) * G[i][j] * Math.min(1, (8 - j) / (24 - i));
    }
    total += pI * inner;
  }
  return total;
}
```

### Step 3 — Bisect for the root

$W_C(0) = 1/3$ by symmetry (all continuous). As $p$ increases, $W_C$ dips well below $1/3$ as discrete robots cluster and cover most races. Then near $p=1$, $W_C$ climbs back above $1/3$ as opponents are nearly all discrete and leave uncovered races. We bisect in $[0.99, 1)$ to find where $W_C$ crosses $1/3$ again.

```javascript
let lo = 0.99, hi = 1 - 1e-12;
for (let iter = 0; iter < 100; iter++) {
  const mid = (lo + hi) / 2;
  if (wc(mid) < 1 / 3) lo = mid; else hi = mid;
}
const p = (lo + hi) / 2;  // ≈ 0.99956044
```

100 iterations gives ~30 digits of precision — far more than the 6 significant figures required.

---

<div style="background:#f6f8fa;border:1px solid #d0d7de;border-radius:6px;padding:1.2em;margin:1.5em 0">
  <button onclick="solveRobotPuzzle()" style="background:#0969da;color:#fff;border:none;padding:0.4em 1.1em;border-radius:4px;cursor:pointer;font-size:0.9em">Run solver</button>
  <pre id="robot-output" style="margin:1em 0 0 0;font-size:0.85em;white-space:pre-wrap;color:#24292f;background:none;border:none;padding:0"></pre>
</div>

<script>
function solveRobotPuzzle() {
  const G = Array.from({length: 24}, () => new Float64Array(9));
  G[0][0] = 1;
  for (let i = 1; i <= 23; i++) {
    for (let j = 1; j <= Math.min(i, 8); j++) {
      G[i][j] = (j / 8) * (G[i - 1][j] + G[i - 1][j - 1]);
    }
  }

  function binom(n, k) {
    if (k < 0 || k > n) return 0;
    let r = 1;
    for (let i = 0; i < k; i++) r = r * (n - i) / (i + 1);
    return r;
  }

  function wc(p) {
    let total = 0;
    for (let i = 0; i <= 23; i++) {
      const pI = binom(23, i) * Math.pow(p, i) * Math.pow(1 - p, 23 - i);
      if (pI < 1e-300) continue;
      let inner = 0;
      for (let j = 0; j <= Math.min(i, 7); j++) {
        inner += binom(8, j) * G[i][j] * Math.min(1, (8 - j) / (24 - i));
      }
      total += pI * inner;
    }
    return total;
  }

  let lo = 0.99, hi = 1 - 1e-12;
  for (let iter = 0; iter < 100; iter++) {
    const mid = (lo + hi) / 2;
    if (wc(mid) < 1 / 3) lo = mid; else hi = mid;
  }
  const p = (lo + hi) / 2;

  document.getElementById('robot-output').textContent = [
    'Sanity checks:',
    `  W_C(0.000) = ${wc(0).toFixed(8)}  ← should equal 1/3 by symmetry`,
    `  W_C(0.500) = ${wc(0.5).toFixed(8)}  ← well below 1/3`,
    `  W_C(0.990) = ${wc(0.99).toFixed(8)}  ← approaching 1/3 from below`,
    `  W_C(1.000) = ${wc(1 - 1e-12).toFixed(8)}  ← above 1/3`,
    '',
    'Bisecting for W_C(p) = 1/3 in [0.99, 1)...',
    '',
    `  p      = ${p.toFixed(8)}`,
    `  W_C(p) = ${wc(p).toFixed(8)}`,
  ].join('\n');
}
</script>

---

## Resources

- [Jane Street Puzzle Statement](https://www.janestreet.com/puzzles/robot-updated-swimming-trials-index/)
- [Official Solution](https://www.janestreet.com/puzzles/robot-updated-swimming-trials-solution/)
