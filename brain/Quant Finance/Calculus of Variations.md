---
tags: [quant-finance, calculus, calculus-of-variations, euler-lagrange, optimal-execution]
---
# Calculus of Variations

Ordinary calculus finds the *number* that minimizes a function. Calculus of variations finds the *whole path* (a function) that minimizes a cost. In finance this is how you pick an optimal trading schedule, an optimal consumption path, or an optimal hedge over time. The main result is the Euler-Lagrange equation, derived below from scratch.

---

## FROM FUNCTIONS TO FUNCTIONALS

An ordinary function takes a number and returns a number: $f(x)$. You minimize it by solving $f'(x) = 0$.

A **functional** takes a whole function and returns a number. The input is a curve $x(t)$, and the output is a single cost $J[x]$. Square brackets are the convention for "this eats a function".

Example: you must sell $X$ shares by time $T$. Any schedule $x(t)$ (shares still held at time $t$) has a total cost

$$
J[x] = \int_0^T \Big[\eta\, \dot{x}(t)^2 + \lambda\sigma^2\, x(t)^2 - \alpha(t)\, x(t)\Big]\, dt
$$

where $\dot{x}$ is the trading speed. Every possible schedule gets a score, and we want the schedule with the lowest one. The unknown is a function, not a number.

Throughout, I write the argument $x(t)$ when defining it and drop the $(t)$ in equations where every term is at the same instant.

### THE GENERAL SHAPE

Most problems look like

$$
J[x] = \int_a^b L\big(t, x, \dot{x}\big)\, dt, \qquad x(a) = x_a, \quad x(b) = x_b
$$

$L$ is called the **Lagrangian**: the cost per unit of time, as a function of time, position and speed. The endpoints are fixed (you start at $X$ and must end at $0$).

---

## THE IDEA: WIGGLE THE PATH

For an ordinary function, a minimum means no small move to the left or right helps. Do the same with a path:

1. Suppose $x$ is the best path.
2. Nudge it to a nearby path $x + \varepsilon h$, where $h(t)$ is any wiggle shape and $\varepsilon$ is a small number controlling its size.
3. The endpoints must stay fixed, so the wiggle must vanish at both ends: $h(a) = h(b) = 0$.
4. The cost of the nudged path is an ordinary function of the single number $\varepsilon$:

$$
\varphi(\varepsilon) = J[x + \varepsilon h]
$$

If $x$ is the minimizer, $\varphi$ has its minimum at $\varepsilon = 0$, so $\varphi'(0) = 0$. This must hold for **every** wiggle $h$.

![[calculus-of-variations-perturbation.png]]

Left: the best path (blue) and two nudged versions that start and end at the same points. Right: the cost as the nudge size $\varepsilon$ varies. Every nudge, in either direction, makes the cost worse, and $\varepsilon = 0$ is the bottom of the bowl.

---

## DERIVING EULER-LAGRANGE

### STEP 1: DIFFERENTIATE WITH RESPECT TO THE NUDGE

$$
\varphi(\varepsilon) = \int_a^b L\big(t,\; x + \varepsilon h,\; \dot{x} + \varepsilon \dot{h}\big)\, dt
$$

The bounds $a$, $b$ do not depend on $\varepsilon$, so we can differentiate inside the integral (the constant-bounds case of [[Leibniz Integral Rule]]). Use the chain rule: $x + \varepsilon h$ changes by $h$ per unit of $\varepsilon$, and $\dot{x} + \varepsilon \dot{h}$ changes by $\dot{h}$. At $\varepsilon = 0$:

$$
\varphi'(0) = \int_a^b \left[\frac{\partial L}{\partial x}\, h + \frac{\partial L}{\partial \dot{x}}\, \dot{h}\right] dt = 0
$$

This is the **first variation**. It says how the cost changes, to first order, when you nudge the path by $h$.

### STEP 2: GET RID OF $\dot{h}$ WITH INTEGRATION BY PARTS

The first term has $h$, the second has its derivative $\dot{h}$. We want everything multiplied by the same $h$. Integration by parts says $\int u\, \dot{h}\, dt = [u\, h] - \int \dot{u}\, h\, dt$. With $u = \partial L / \partial \dot{x}$:

$$
\int_a^b \frac{\partial L}{\partial \dot{x}}\, \dot{h}\, dt = \underbrace{\left[\frac{\partial L}{\partial \dot{x}}\, h\right]_a^b}_{=\,0} - \int_a^b \frac{d}{dt}\!\left(\frac{\partial L}{\partial \dot{x}}\right) h\, dt
$$

The boundary term is zero because $h(a) = h(b) = 0$. This is exactly where "endpoints are fixed" is used. Substituting back:

$$
\int_a^b \left[\frac{\partial L}{\partial x} - \frac{d}{dt}\frac{\partial L}{\partial \dot{x}}\right] h\, dt = 0 \quad \text{for every wiggle } h
$$

### STEP 3: THE BRACKET MUST BE ZERO

If an integral of $g \cdot h$ vanishes for every allowed $h$, then $g$ itself must be zero everywhere. Reason: suppose $g$ were positive somewhere. Choose a bump $h$ that is positive only near that spot and zero elsewhere. Then $\int g\, h\, dt > 0$, a contradiction. The same argument rules out $g$ negative anywhere. (This is the **fundamental lemma of the calculus of variations**.)

### RESULT

$$
\boxed{\;\frac{\partial L}{\partial x} - \frac{d}{dt}\,\frac{\partial L}{\partial \dot{x}} = 0\;}
$$

This is the **Euler-Lagrange equation**. It turns "find the best function" into a differential equation for $x$, together with the endpoint conditions.

### HOW TO USE IT

1. Write down $L$.
2. Compute $\partial L / \partial x$ (treat $\dot{x}$ as a separate variable).
3. Compute $\partial L / \partial \dot{x}$, then take $d/dt$ of it (now $x$ and $\dot{x}$ depend on $t$, so use the chain rule).
4. Subtract, set to zero, solve with the endpoint conditions.

---

## EXAMPLES

### SHORTEST PATH IS A STRAIGHT LINE

The length of a curve $y(x)$ from one point to another is $\int \sqrt{1 + y'^2}\, dx$, so $L = \sqrt{1 + y'^2}$. It does not depend on $y$, so $\partial L / \partial y = 0$, and the equation becomes

$$
\frac{d}{dx}\,\frac{y'}{\sqrt{1 + y'^2}} = 0
$$

The slope is constant, so the path is a straight line.

### OPTIMAL EXECUTION WITH ALPHA

Take $L = \eta \dot{x}^2 + \lambda\sigma^2 x^2 - \alpha x$. Then

$$
\frac{\partial L}{\partial x} = 2\lambda\sigma^2 x - \alpha, \qquad \frac{\partial L}{\partial \dot{x}} = 2\eta \dot{x} \;\Rightarrow\; \frac{d}{dt}\frac{\partial L}{\partial \dot{x}} = 2\eta \ddot{x}
$$

Euler-Lagrange gives $2\lambda\sigma^2 x - \alpha - 2\eta \ddot{x} = 0$. Divide by $2\eta$ and let $\kappa^2 = \lambda\sigma^2/\eta$:

$$
\ddot{x} = \kappa^2 x - \frac{\alpha}{2\eta}
$$

Reading it as a force balance:

- $\kappa^2 x$ is the risk push. It is large when the position is large, so selling slows down as the position shrinks (acceleration $> 0$).
- $-\alpha / (2\eta)$ is the alpha push. Positive alpha makes it negative, so you hold longer and sell later.
- Trading cost $\eta$ acts like mass: the more expensive it is to trade, the harder it is to change pace.

With $\alpha = 0$ and $x(0) = X$, $x(T) = 0$ the solution is

$$
x(t) = X\, \frac{\sinh \kappa (T - t)}{\sinh \kappa T}
$$

### DISCRETE VERSION OF THE SAME IDEA

Split $[0, T]$ into buckets. Each holding $x_k$ appears in exactly four places in the cost: the trade into bucket $k$, the trade out of it, its own risk, its own alpha. Setting $\partial J / \partial x_k = 0$ is ordinary calculus. Integration by parts in the continuous derivation plays the role of "$x_k$ appears in two neighboring trades". Euler-Lagrange is the limit of that bucket condition as the buckets shrink.

---

## IS IT A MINIMUM?

Euler-Lagrange is a **necessary** condition: a minimizer must satisfy it, as setting $f'(x) = 0$ is for a number. It does not by itself say the solution is a minimum (it could be a maximum or saddle).

For the execution problem it is also **sufficient**. $J[x + \varepsilon h]$ is quadratic in $\varepsilon$ with curvature

$$
\varphi''(0) = 2\int_0^T \left[\eta\, \dot{h}^2 + \lambda\sigma^2 h^2\right] dt \;\ge\; 0
$$

for $\eta > 0$ and $\lambda\sigma^2 \ge 0$. The bowl in the right panel of the chart always opens upward, so a path that satisfies Euler-Lagrange with the endpoint conditions is the global minimum. It is unique when $\eta > 0$.

---

## SUMMARY

| Ordinary calculus                         | Calculus of variations                                    |
|---|---|
| Unknown is a number $x$                   | Unknown is a function $x(t)$                              |
| Cost is a function $f(x)$                 | Cost is a functional $J[x]$                               |
| Condition: $f'(x) = 0$                    | Condition: $\partial_x L - \frac{d}{dt}\partial_{\dot{x}} L = 0$ |
| Test with a small step $x + \varepsilon$  | Test with a small wiggle $x + \varepsilon h$              |
| Result: an equation                       | Result: a differential equation                           |

Related: [[Concave Functions and Their Derivatives]].
