---
tags: [quant-finance, derivatives, option-pricing, risk-neutral, probability]
---
# Risk-Neutral Pricing

An option's price equals its discounted expected payoff under the risk-neutral measure $\mathbb{Q}$. The formula is model-independent: it holds for any risk-neutral density, and Black-Scholes is just the special case where that density is lognormal.

---

## THE PRICING FORMULA

For a European call with strike $K$ and maturity $T$:

$$
C = e^{-rT}\, \mathbb{E}^{\mathbb{Q}}\!\left[(S_T - K)^+\right]
$$

Since the payoff $(S_T - K)^+$ is zero when $S_T < K$, writing the expectation as an integral against the density of $S_T$ gives:

$$
C(K) = e^{-rT} \int_K^\infty (S - K)\, q(S)\, dS
$$

Assumptions baked in:

- $q(S)$ must be the **risk-neutral** density of $S_T$, not the real-world density. Plugging in a distribution estimated from historical returns gives the wrong price.
- $r$ is constant (or at least deterministic), which is what lets $e^{-rT}$ sit outside the integral. With stochastic rates correlated to the asset, switch to the $T$-forward measure.

### BREEDEN-LITZENBERGER

The relationship can be inverted. The risk-neutral density is recovered from the curvature of call prices across strikes:

$$
q(K) = e^{rT}\, \frac{\partial^2 C}{\partial K^2}
$$

The pricing integral and this inversion are two sides of the same coin.

---

## WHAT RISK-NEUTRAL MEANS

The name is misleading: it is not a claim that investors don't care about risk. It is a pricing trick.

Risky payoffs need a risk premium, and computing the right premium directly is hard. Instead of adjusting for risk in the discount rate while using real-world probabilities, fold the entire risk adjustment into the probabilities and discount at the plain risk-free rate. Under these adjusted probabilities $\mathbb{Q}$, every asset earns $r$, as if everyone were neutral to risk.

These probabilities are not arbitrary. No-arbitrage forces a unique set of them to exist (in a complete market), pinned down by the prices of assets already trading. Pricing under $\mathbb{Q}$ means pricing consistently with everything else in the market.

### ONE-STEP EXAMPLE

A stock is at $\$100$ and moves to $\$120$ or $\$80$ in one step, with $r = 0$. Solve for the $p$ that makes the expected value equal the current price:

$$
100 = p \cdot 120 + (1-p) \cdot 80 \;\Rightarrow\; p = 0.5
$$

Any derivative on this stock is priced by taking its expected payoff with $p = 0.5$ and discounting at $r$. This $p$ says nothing about how likely the stock really is to rise; it is the number that keeps prices arbitrage-free.

---

## DIVIDENDS AND CARRY

Dividends (and other carry: storage, convenience yield, foreign rates) don't appear in the pricing formula. They live inside $q(S)$.

Under $\mathbb{Q}$, the discounted price of a traded asset is a martingale. With a continuous dividend yield $\delta$, total return (price + dividends) must earn $r$, so price appreciation alone earns $r - \delta$:

$$
\mathbb{E}^{\mathbb{Q}}[S_T] = S_0\, e^{(r - \delta)T}
$$

Dividends shift the density downward. The discount factor $e^{-rT}$ is unchanged, since it discounts a cash payoff at the risk-free rate.

### BLACK-SCHOLES WITH DIVIDENDS

The lognormal $q$ has mean log $\ln S_0 + (r - \delta - \tfrac{1}{2}\sigma^2)T$, giving:

$$
C = S_0 e^{-\delta T} N(d_1) - K e^{-rT} N(d_2)
$$

$$
d_1 = \frac{\ln(S_0/K) + (r - \delta + \tfrac{1}{2}\sigma^2)T}{\sigma\sqrt{T}}, \quad d_2 = d_1 - \sigma\sqrt{T}
$$

The $e^{-\delta T}$ and the $-\delta$ in $d_1$ both trace back to the shifted density, not to any change in the pricing integral.

### THE FORWARD PRICE

Every kind of carry collapses into the forward price $F$. The pricing formula only needs $F$ and the discount factor; each asset class computes $F$ its own way:

| Asset                                                  | Forward                                       |
|---|---|
| Equity index, continuous yield $\delta$                | $F = S_0 e^{(r-\delta)T}$                     |
| Single stock, discrete dividends                       | $F = \left(S_0 - \text{PV(divs)}\right)e^{rT}$ |
| FX, foreign rate $r_f$ (Garman-Kohlhagen)              | $F = S_0 e^{(r - r_f)T}$                      |
| Commodity, storage cost $u$, convenience yield $y$     | $F = S_0 e^{(r + u - y)T}$                    |

---

## REAL-WORLD VS RISK-NEUTRAL PROBABILITIES

The two measures come from different sources. Neither is derived from the other directly.

- **Risk-neutral $\mathbb{Q}$** is **calibrated** from current market prices. The one-step example used only prices, never historical frequencies. In practice, invert observed option prices across strikes (Breeden-Litzenberger). It is forward-looking and embeds the market's view and its risk aversion, tangled together.
- **Real-world $\mathbb{P}$** is **estimated** from historical returns (or your own beliefs). Fit drift and volatility to a return series, or overlay a forecast. It answers what is actually likely to happen.

|                | $\mathbb{Q}$ (risk-neutral)  | $\mathbb{P}$ (real-world)        |
|---|---|---|
| Source         | Current market prices        | Historical data / beliefs        |
| Method         | Calibration (invert prices)  | Estimation (fit to returns)      |
| Drift of stock | Forced to be $r - \delta$    | Whatever the data says (e.g. 8%) |
| Answers        | What must prices be?         | What is likely to happen?        |

The drift is the clearest distinction. A biotech and a utility share the same risk-neutral drift $r - \delta$ but have very different real-world drifts. The gap between the true expected return and $r - \delta$ is the **risk premium**.

Option prices only reveal $\mathbb{Q}$; a price chart only reveals $\mathbb{P}$. Bridging them (estimating the risk premium or pricing kernel) is a hard, actively researched problem, related to the equity risk premium puzzle.
