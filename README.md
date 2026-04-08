# Quantum Walk–Based Stock Option Pricing
### A Comparison with Classical Dynamic Programming

**Team: Schrodinger's Coders**
Swapnil Bhole · Varun Srinivasan · Aaron Sequeria · Aditya Senthil · Michael Shio

---

## Overview

This project asks a fundamental question in quantitative finance: **when is the right moment to exercise an American-style stock option?** We solve the same problem two ways — once with a mathematically exact classical solution, and once by replacing the classical stochastic model with a quantum walk running on a quantum circuit simulator — and then compare the results head to head.

The core finding: **the quantum walk approach achieves a slightly higher mean profit (0.05567 vs 0.05533) while recovering nearly identical exercise behavior**, validating quantum walks as a viable alternative kernel for stochastic dynamic programming.

---

## The Problem

You hold an option to **buy one share of a stock at a fixed strike price K**. The stock price evolves day by day as a random walk:

```
X_{k+1} = X_k + W_k,    k = 0, 1, 2, ...
```

where each `W_k` is drawn i.i.d. from `Exp(2) - 0.5` — an exponential distribution with rate 2, shifted left by 0.5, giving a mean increment of zero. You have **N = 2 days** to decide. You can exercise **at most once**, and if you exercise on day `k` when the price is `x`, your profit is `x - K`. You can also walk away with nothing.

**Parameters:**

| Parameter | Value |
|-----------|-------|
| Strike price K | 2.0 |
| Initial price X0 | 1.0 |
| Noise distribution | Exp(2) - 0.5 |
| Days N | 2 |

**Goal: find the strategy that maximizes expected profit.**

---

## Approach 1: Classical Dynamic Programming (Exact)

### The Bellman Equations

Define `V_t(x)` = maximum expected profit when the current price is `x` and there are `t` days remaining:

```
V_2(x) = max(x - K, 0)                        <- terminal: exercise or walk away
V_1(x) = max(x - K, E_W[V_2(x + W)])          <- one day left
V_0(x) = max(x - K, E_W[V_1(x + W)])          <- two days left
```

### Structural Guarantees (Proven Analytically)

Two key properties of the value function can be proven by induction:

- **`V_t(x) - x` is non-increasing in x** — the benefit of waiting diminishes as the price rises, so there is a unique crossover point where exercising now beats waiting.
- **`V_t(x) >= V_{t+1}(x)` for all x** — more time remaining is always at least as valuable.

Together, these guarantee that the **optimal policy is always a threshold policy**: at each time step `t`, there exists a number `p_t` such that you exercise if and only if `x >= p_t`, and the thresholds decrease over time:

```
p_0 >= p_1 >= ... >= p_{N-1} >= p_N = K
```

### Closed-Form Solution

For `W ~ Exp(2) - 0.5`, the analytical solution (derived using the memoryless property of the exponential) gives:

```
p_i = K + (N - i) / 2
```

| Time step | Threshold | Interpretation |
|-----------|-----------|----------------|
| p_2 = K | **2.0** | Exercise at expiry if profitable |
| p_1 | **2.5** | Exercise on day 1 only if price >= 2.5 |
| p_0 | **3.0** | Exercise on day 0 only if price >= 3.0 |

Starting from `X_0 = 1`, the exact optimal expected reward is `V_0(1) = 3e^{-4} ≈ 0.0549`. Note: despite a 63.2% chance of the price dropping each day, the patient strategy yields positive expected profit.

---

## Approach 2: Quantum Walk DP

Instead of deriving transition probabilities analytically, we build a **variational quantum circuit** that simulates one step of the stock price random walk. The same DP Bellman equations are then applied on top of the quantum-sampled transition matrix — same framework, different model of how prices move.

### Circuit Architecture

**Price discretization:** The price range `[-1, 6]` is divided into 16 bins.

**Quantum registers:**
- **1 coin qubit** — controls direction (`|0>` = price down, `|1>` = price up)
- **16 position qubits** — unary encoding: exactly one qubit is `|1>`, indicating the current price bin

**One step of the walk, U = S(C x I):**

```python
# Coin operator: parameterized rotation (theta = pi/2 for symmetric walk)
qc.ry(theta, 0)

# Shift operator: controlled swaps move the active bin left or right
# Right move (coin == |1>)
for i in reversed(range(n_bins - 1)):
    cswap_between(qc, 0, 1 + i, 1 + (i + 1))

# Left move (coin == |0>)
qc.x(0)
for i in range(n_bins - 1):
    cswap_between(qc, 0, 1 + i, 1 + (i + 1))
qc.x(0)

# Quantum interference: CZ gates between adjacent position bins
for i in range(n_bins - 1):
    qc.cz(1 + i, 1 + i + 1)
```

The CZ gates between adjacent position qubits introduce **quantum interference**, causing the resulting probability distribution to differ structurally from the classical exponential. This is the core mechanism that distinguishes the quantum model.

### Building the Transition Matrix

The circuit is run from each of the 16 starting bins (400 shots per bin) to construct a full 16x16 transition matrix `T_q`, where `T_q[b, b']` = probability of moving from price bin `b` to bin `b'` in one step.

### DP on the Quantum Matrix

Standard value iteration is applied to `T_q` over the discrete price grid:

```python
V2_bins = max(bin_centers - K, 0)
EV2     = T_q @ V2_bins
p1_est  = first bin where (bin_center - K) >= EV2[bin]   # indifference point

V1_bins = max(bin_centers - K, EV2)
EV1     = T_q @ V1_bins
p0_est  = first bin where (bin_center - K) >= EV1[bin]
```

---

## Head-to-Head Comparison

### Probability Spread (1 step from X_0 = 1.0)

The classical exponential walk concentrates nearly all probability near price shift 0 and decays rapidly. The quantum walk's interference redistributes probability differently across shifts — creating a structurally distinct transition kernel that feeds into the DP and leads to different (slightly more aggressive) exercise thresholds.

The deeper contrast appears over many steps: after 500 steps, the classical walk (red) produces a narrow Gaussian centered at 0, while the quantum walk (blue) produces a **bimodal distribution** with probability mass concentrated near the extremes — a direct consequence of quantum interference suppressing the center and amplifying the tails.

### Exercise Thresholds

Both approaches recover the same threshold structure (`p_0 >= p_1 >= p_2 = K`). Quantum thresholds closely match the classical analytical values. The quantum DP is slightly more aggressive early — it exercises at marginally lower prices on day 0 — reflecting the different tail weights in its transition model.

### Simulation Results (50,000 trials, fixed seed)

| Feature | Classical DP | Quantum Walk DP | Notes |
|---------|-------------|-----------------|-------|
| Threshold calculation | Deterministic, exact | Estimated via quantum sampling | Quantum approximates classical well |
| p_0 (day 0 threshold) | 3.0 (analytical) | ~3.0 (quantum approx) | Closely matched |
| p_1 (day 1 threshold) | 2.5 (analytical) | ~2.5 (quantum approx) | Closely matched |
| **Mean profit** | **0.05533** | **0.05567** | Quantum slightly higher (+0.6%) |
| **Std profit** | **0.24336** | **0.24738** | Quantum slightly more risk (+1.6%) |
| **Proportion exercised** | **0.09058** | **0.09052** | Nearly identical |
| Early exercise behavior | Conservative | Slightly aggressive | Quantum exercises at lower prices early |
| Computational cost | Exact, hard to scale | Depends on quantum simulation | Quantum may scale better for large grids |

### What the Numbers Mean

The quantum walk DP achieves **+0.6% higher mean profit** at the cost of **+1.6% higher standard deviation**. This is a modest risk-return tradeoff from the slightly more aggressive early exercise policy. The proportion of trials where the option is exercised (9%) is virtually identical between both approaches, meaning the quantum policy is not simply exercising more often — it is exercising at more favorable moments on average.

### Why the Quantum Walk Can Win

The classical DP is optimal *given that the true distribution is Exp(2) - 0.5*. The quantum circuit does not know this distribution — it constructs its own transition model through sampling and interference. When that model leads to different (slightly lower) exercise thresholds on day 0, the simulated profits happen to be slightly higher, suggesting the quantum model's implicit assumptions about price dynamics are marginally better tuned to the Monte Carlo simulation environment.

### Scalability

The classical approach requires analytical integration over the noise distribution at each Bellman step — tractable for simple distributions like Exp(2) - 0.5, but increasingly difficult for:
- High-dimensional price spaces (multi-asset options)
- Unknown or empirically estimated noise distributions
- Time-varying dynamics

The quantum walk instead **samples the transition structure directly from a circuit**, which could scale more naturally as quantum hardware matures.

---

## Visualizations

**Quantum vs Classical Random Walk (500 steps):** The quantum walk (blue) shows a bimodal distribution with peaks near ±350, while the classical walk (red/pink) produces a narrow Gaussian centered at 0. This is the signature of quantum interference: probability is suppressed at the center and amplified at the extremes.

**Quantum vs Classical Walk Probability Spread (1 step):** Transition probabilities from X_0 aggregated into 5 price-shift buckets. Classical decays exponentially; quantum is reshaped by interference.

**Value Functions and Thresholds:** Smooth analytical curves for V_2(x) (terminal payoff, blue) and V_1(x) (one-period value, orange), with classical thresholds marked `x` and quantum-derived thresholds marked `o`. Shows that holding the option is worth more than immediate exercise across most of the price range.

---

## Files

| File | Description |
|------|-------------|
| [QuantUM_hackathon.ipynb](QuantUM_hackathon.ipynb) | Main notebook — quantum circuits, classical DP, simulation, plots |
| [Dynamic_prog_vs quantum_walk.pdf](Dynamic_prog_vs%20quantum_walk.pdf) | Project presentation slides with full results table |
| [README.md](README.md) | This file |

---

## Setup

```bash
pip install qiskit qiskit-aer numpy scipy matplotlib pandas
```

Open `QuantUM_hackathon.ipynb` in Jupyter or VS Code and run all cells in order. Cell 1 runs the full pipeline (quantum circuit, DP, Monte Carlo simulation). Cell 2 generates the comparison plots.


---

## Key Takeaway

The quantum walk does not know the true distribution of `W`. It constructs a transition model purely from quantum sampling and interference — yet recovers thresholds close to the analytically optimal ones and achieves a slightly higher simulated profit. This demonstrates that **quantum walks can serve as a drop-in replacement for the stochastic transition model** in dynamic programming, opening a path toward quantum-enhanced financial optimization on future hardware.
