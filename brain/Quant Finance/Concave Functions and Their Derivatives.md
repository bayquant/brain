---
tags: [quant-finance, calculus, concavity, convexity, derivatives]
---
# Concave Functions and Their Derivatives

If $f$ is concave and you sketch its derivative $f'$, how do you tell whether $f'$ is convex or concave? Concavity of $f$ only tells you that $f'$ is **decreasing**. The shape of $f'$ depends on the third derivative.

---

## THE RULE

- $f$ concave $\iff f'' \le 0 \iff f'$ is decreasing.
- $f'$ convex $\iff f''' \ge 0$: the slope of $f$ shrinks fast, then levels off.
- $f'$ concave $\iff f''' \le 0$: the slope of $f$ shrinks slowly, then collapses.

---

## HOW TO READ IT OFF THE CHART

Take an increasing, concave $f$ (a quantity that keeps growing, but at a slower rate). Look at how the tangent slope changes as you move right.

| Shape of $f$                               | Slope of $f$ over time    | Shape of $f'$ | Example                 |
|---|---|---|---|
| Flattens out (diminishing returns)         | Falls fast, then levels   | Convex        | $\sqrt{x}$, $\log x$    |
| Bends over faster and faster (hilltop)     | Falls slowly, then drops  | Concave       | $\sin x$ on $[0,\pi/2]$ |

![[concave-function-derivative-shapes.png]]

---

## CHORD TEST

Pick two points $a < c$ on the graph of $f'$ and draw the chord between them (dashed in the chart).

- Curve **below** the chord: $f'$ is convex.
- Curve **above** the chord: $f'$ is concave.

---

## WORKED EXAMPLES

$$
f(x) = \sqrt{x}: \quad f'(x) = \tfrac{1}{2} x^{-1/2}, \quad f'''(x) = \tfrac{3}{8} x^{-5/2} > 0 \;\Rightarrow\; f' \text{ convex}
$$

$$
f(x) = \sin x: \quad f'(x) = \cos x, \quad f'''(x) = -\cos x \le 0 \text{ on } [0, \pi/2] \;\Rightarrow\; f' \text{ concave}
$$
