---
tags: [quant-finance, calculus, leibniz-rule, differentiation-under-the-integral]
---
# Leibniz Integral Rule

The Leibniz integral rule (differentiation under the integral sign) gives the derivative of an integral whose integrand and bounds both depend on a parameter $t$. In finance it is the tool behind results like Breeden-Litzenberger, where you differentiate an option price with respect to a strike that appears in the integration bound.

Reference: [Full Leibniz Rule with Moving Bounds](https://www.youtube.com/watch?v=vAJjFuhVmEo) (The Feynman Technique).

---

## THE RULE

For

$$
F(t) = \int_{a(t)}^{b(t)} f(x, t)\, dx
$$

the derivative is

$$
F'(t) = \underbrace{f\big(b(t), t\big)\, b'(t)}_{\text{upper bound moves}} \;-\; \underbrace{f\big(a(t), t\big)\, a'(t)}_{\text{lower bound moves}} \;+\; \underbrace{\int_{a(t)}^{b(t)} \frac{\partial f}{\partial t}(x, t)\, dx}_{\text{integrand changes}}
$$

Conditions: $f$ and $\partial f / \partial t$ are continuous in both $x$ and $t$, and $a(t)$, $b(t)$ are differentiable.

### SPECIAL CASES

- **Constant bounds** ($a' = b' = 0$): the boundary terms drop out, leaving the familiar $F'(t) = \int_a^b \partial_t f\, dx$.
- **Integrand independent of $t$**, lower bound constant: $F'(t) = f(b(t))\, b'(t)$, which is the fundamental theorem of calculus combined with the chain rule.

---

## WHAT IT MEANS: HOW THINGS MOVE

$F(t)$ is the area under the curve $f(\cdot, t)$ between two vertical walls at $a(t)$ and $b(t)$. As $t$ changes, three things happen at once: both walls slide, and the curve itself reshapes.

![[leibniz-snapshots.png]]

Over a small step $\Delta t$, the change in area splits into three pieces, one per term of the rule:

![[leibniz-decomposition.png]]

- **Green strip (upper bound):** the right wall moves by $b'(t)\Delta t$. The new sliver has height $\approx f(b, t)$, so it adds $f(b, t)\, b'(t)\, \Delta t$.
- **Red strip (lower bound):** the left wall moves by $a'(t)\Delta t$. Moving it right removes a sliver of height $\approx f(a, t)$, hence the minus sign: $-f(a, t)\, a'(t)\, \Delta t$.
- **Blue band (interior):** over the original interval, each point of the curve moves vertically by $\approx \partial_t f\, \Delta t$. Summed over the interval this is $\Delta t \int_a^b \partial_t f\, dx$. It is signed: where the curve drops, it subtracts area.

Dividing by $\Delta t$ and letting $\Delta t \to 0$ gives exactly the three terms of $F'(t)$. The strips become infinitely thin rectangles, which is what the mean value theorem step in the proof makes rigorous.

The next chart tracks the three terms over time and checks that their sum matches a finite-difference derivative of $F$ (they agree to about $10^{-7}$):

![[leibniz-terms.png]]

Charts use $f(x,t) = 1.5 + \sin(1.5x - t) + 0.4t$, $a(t) = 0.5 + 0.4t$, $b(t) = 2.5 + 0.8t$.

---

## PROOF STEPS

Following the video's derivation.

### 1. LIMIT DEFINITION

Plug $F$ into the definition of the derivative:

$$
F'(t) = \lim_{h \to 0} \frac{1}{h}\left[\int_{a(t+h)}^{b(t+h)} f(x, t+h)\, dx - \int_{a(t)}^{b(t)} f(x, t)\, dx\right]
$$

### 2. ADD AND SUBTRACT

Add and subtract $\int_{a(t)}^{b(t)} f(x, t+h)\, dx$ inside the brackets. This mixes "new integrand" with "old bounds" so the two effects can be separated.

### 3. SPLIT THE MOVED INTEGRAL

Use additivity of integrals, $\int_{c_1}^{c_4} = \int_{c_1}^{c_2} + \int_{c_2}^{c_3} + \int_{c_3}^{c_4}$, to break the integral over the new bounds into three pieces:

$$
\int_{a(t+h)}^{b(t+h)} = \int_{a(t+h)}^{a(t)} + \int_{a(t)}^{b(t)} + \int_{b(t)}^{b(t+h)}
$$

The middle piece cancels with the subtracted term from step 2.

### 4. FLIP AND REGROUP

Use $\int_{c_1}^{c_2} = -\int_{c_2}^{c_1}$ on the lower-bound piece, then regroup:

$$
F'(t) = \lim_{h \to 0} \frac{1}{h}\left[\int_{b(t)}^{b(t+h)} f(x, t+h)\, dx - \int_{a(t)}^{a(t+h)} f(x, t+h)\, dx + \int_{a(t)}^{b(t)} \big[f(x, t+h) - f(x, t)\big]\, dx\right]
$$

### 5. SPLIT THE LIMIT: INTERIOR TERM

Take the limit of each part separately. Since $a(t)$ and $b(t)$ don't depend on $x$, the limit moves inside the interior integral (this is where continuity is needed), and it becomes the definition of a partial derivative:

$$
\lim_{h \to 0} \int_{a(t)}^{b(t)} \frac{f(x, t+h) - f(x, t)}{h}\, dx = \int_{a(t)}^{b(t)} \frac{\partial f}{\partial t}(x, t)\, dx
$$

### 6. MEAN VALUE THEOREM: BOUNDARY TERMS

The **mean value theorem for integrals**: if $g$ is continuous on $[p, q]$, there is a $c \in [p, q]$ with $\int_p^q g(x)\, dx = g(c)(q - p)$.

Apply it to the tiny upper interval: there is a $c_1 \in [b(t), b(t+h)]$ with

$$
\int_{b(t)}^{b(t+h)} f(x, t+h)\, dx = f(c_1, t+h)\,\big[b(t+h) - b(t)\big]
$$

As $h \to 0$, the interval collapses, so $c_1 \to b(t)$ and $f(c_1, t+h) \to f(b(t), t)$. The same argument on the lower interval gives a $c_2 \in [a(t), a(t+h)]$ with $f(c_2, t+h) \to f(a(t), t)$.

### 7. TAKE THE LIMITS

$f(b(t), t)$ doesn't depend on $h$, so it comes out of the limit, leaving the derivative definitions of $b'$ and $a'$:

$$
\lim_{h \to 0} \frac{b(t+h) - b(t)}{h} = b'(t), \qquad \lim_{h \to 0} \frac{a(t+h) - a(t)}{h} = a'(t)
$$

Putting everything together gives the rule.

---

## WORKED EXAMPLE

$$
F(t) = \int_{t}^{t^2} x t\, dx
$$

Here $f(x, t) = xt$, $a(t) = t$, $b(t) = t^2$, so $a' = 1$, $b' = 2t$, $\partial_t f = x$.

- Upper bound: $f(t^2, t) \cdot 2t = t^3 \cdot 2t = 2t^4$
- Lower bound: $-f(t, t) \cdot 1 = -t^2$
- Interior: $\int_t^{t^2} x\, dx = \tfrac{1}{2}(t^4 - t^2)$

$$
F'(t) = 2t^4 - t^2 + \tfrac{1}{2}t^4 - \tfrac{1}{2}t^2 = \tfrac{5}{2}t^4 - \tfrac{3}{2}t^2
$$

Check directly: $F(t) = t \cdot \tfrac{1}{2}(t^4 - t^2) = \tfrac{1}{2}(t^5 - t^3)$, so $F'(t) = \tfrac{5}{2}t^4 - \tfrac{3}{2}t^2$. ✓

---

## APPLICATION: BREEDEN-LITZENBERGER

From [[Risk-Neutral Pricing]], a call price as a function of strike is

$$
C(K) = e^{-rT} \int_K^\infty (S - K)\, q(S)\, dS
$$

Here the parameter is $K$, which appears both in the lower bound and in the integrand.

**First derivative.** Upper bound is constant (no term). Lower bound term: $-(K - K)\, q(K) \cdot 1 = 0$, since the payoff vanishes at the strike. Interior: $\partial_K (S - K) = -1$. So

$$
\frac{\partial C}{\partial K} = -e^{-rT} \int_K^\infty q(S)\, dS = -e^{-rT}\, \mathbb{Q}(S_T > K)
$$

**Second derivative.** Now the integrand $q(S)$ doesn't depend on $K$, so only the lower bound term survives: $-e^{-rT} \cdot \big(-q(K)\big)$. Therefore

$$
\frac{\partial^2 C}{\partial K^2} = e^{-rT}\, q(K) \quad\Longrightarrow\quad q(K) = e^{rT}\, \frac{\partial^2 C}{\partial K^2}
$$

The first derivative also gives a useful by-product: the (discounted) risk-neutral probability of finishing in the money is minus the slope of the call price in strike.
