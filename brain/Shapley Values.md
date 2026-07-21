---
tags: [game-theory, machine-learning, statistics, interpretability]
---
# Shapley Values

A Shapley value is a way to fairly split the total payoff of a cooperative game among its players, based on each player's average marginal contribution across every possible coalition. In machine learning, the same math (as SHAP) attributes a model's prediction across its input features.

---

## CORE CONCEPT

Given a set of players $N$ and a value function $v(S)$ that returns the payoff of any coalition $S \subseteq N$, the Shapley value of player $i$ is:

$$
\varphi_i = \sum_{S \subseteq N \setminus \{i\}} \frac{|S|!\,(|N| - |S| - 1)!}{|N|!} \left[ v(S \cup \{i\}) - v(S) \right]
$$

Read informally: for every possible coalition that excludes player `i`, measure how much adding `i` changes the payoff, then average that marginal contribution over all coalitions and all orderings in which `i` could join.

---

## PROPERTIES (AXIOMS)

Shapley values are the *unique* allocation satisfying all four:

- **Efficiency** — the values sum exactly to the total payoff: $\sum_i \varphi_i = v(N)$
- **Symmetry** — two players who contribute equally to every coalition get equal values
- **Dummy (null player)** — a player who adds zero marginal value to every coalition gets $\varphi_i = 0$
- **Additivity** — for two games combined ($v = v_1 + v_2$), the Shapley values add: $\varphi_i(v) = \varphi_i(v_1) + \varphi_i(v_2)$

These four axioms are what make the allocation "fair" in a provable sense, not just a heuristic.

---

## COMPUTATION

Exact computation requires evaluating $v(S)$ for every subset — $2^n$ coalitions — so it's only tractable for small $n$. Brute force over all orderings (equivalent formulation):

```python
import itertools


def shapley_value(players, value_fn):
    n = len(players)
    contributions = {p: 0.0 for p in players}
    for perm in itertools.permutations(players):
        coalition = set()
        prev_value = value_fn(coalition)
        for p in perm:
            coalition.add(p)
            new_value = value_fn(coalition)
            contributions[p] += new_value - prev_value
            prev_value = new_value
    n_perms = len(list(itertools.permutations(players)))
    return {p: total / n_perms for p, total in contributions.items()}
```

For larger `n`, exact computation is replaced with Monte Carlo sampling of random orderings, or model-specific shortcuts (see below).

---

## SHAP (SHAPLEY ADDITIVE EXPLANATIONS)

SHAP applies Shapley values to model interpretability: treat each input feature as a "player," and the model's prediction (versus its baseline/expected output) as the payoff to explain.

```
players = feature values for one prediction
v(S)    = expected model output when only features in S are "known"
          (others are marginalized out / replaced with background values)
```

$\varphi_i$ is feature $i$'s contribution to this specific prediction:

$$
\text{prediction} = \text{baseline} + \sum_i \varphi_i
$$

Because of the **efficiency** axiom, the feature attributions always sum exactly to $\text{prediction} - \text{baseline}$ — this is what distinguishes SHAP from ad hoc importance scores (like raw feature weights or split-based importance) that don't guarantee this decomposition.

Practical variants avoid the $2^n$ blowup:

| Variant | Applies to | Approach |
|---|---|---|
| KernelSHAP | any model | weighted linear regression over sampled coalitions |
| TreeSHAP | tree ensembles (XGBoost, LightGBM, random forest) | exact, polynomial-time via tree structure |
| DeepSHAP | neural networks | backpropagation-based approximation |
| LinearSHAP | linear models | closed form from coefficients × (value - mean) |

```python
import shap

explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X)   # per-feature, per-row contributions
```

---

## EXAMPLE

Three features `A`, `B`, `C` predicting a value, baseline (empty coalition) = 10:

```
v({})        = 10
v({A})       = 14
v({B})       = 11
v({C})       = 10
v({A,B})     = 18
v({A,C})     = 15
v({B,C})     = 12
v({A,B,C})   = 20
```

Averaging marginal contributions of $A$ over all 6 orderings gives $\varphi_A$, and similarly for $B$, $C$. By construction:

$$
\varphi_A + \varphi_B + \varphi_C = v(\{A,B,C\}) - v(\varnothing) = 10
$$

---

## COMMON PITFALLS

- Treating SHAP values as causal effects — they explain the model's behavior relative to a baseline, not real-world cause and effect
- Ignoring feature correlation — Shapley's "marginalize out the rest" assumption can produce misleading attributions when features are highly correlated (splits credit between correlated features somewhat arbitrarily)
- Choosing an uninformative background/baseline dataset for KernelSHAP — the baseline defines what "0 contribution" means, so a bad choice skews every attribution
- Confusing global vs. local importance — per-prediction SHAP values (local) must be aggregated (e.g. mean absolute value) to get a global feature-importance ranking
- Assuming exact computation scales — brute force is $O(2^n \cdot n)$; anything beyond a handful of features needs TreeSHAP, KernelSHAP sampling, or another approximation
