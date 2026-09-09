---
tags: [linear-algebra, graph-theory, network-analysis, eigenvalues]
---
# Eigenvector Centrality

Eigenvector centrality scores a node by the importance of its neighbors, not just their count — a node is central if it's connected to other central nodes, recursively. It's the dominant eigenvector of the graph's adjacency matrix, and it strictly generalizes degree centrality (which only counts *how many* neighbors, not *how important* they are).

---

## THE RECURSIVE DEFINITION

Want a score $x_i$ per node such that: *my score is proportional to the sum of my neighbors' scores.* In matrix form, one round of "everyone hands their score to their neighbors and sums it up" is $Ax$ — for node $i$, $(Ax)_i = \sum_{j \sim i} x_j$, exactly the sum over $i$'s neighbors.

Requiring self-consistency up to scale gives:

$$
x = \frac{1}{\lambda}Ax \quad\Longleftrightarrow\quad Ax = \lambda x
$$

So $x$ must be an eigenvector of $A$, and $\lambda$ its eigenvalue — this isn't a modeling choice, it falls straight out of writing "importance depends on neighbors' importance" as an equation. The scale factor $\lambda$ is needed because only the *relative* proportions of $x$ across nodes matter, not its absolute magnitude.

---

## WHY THE LARGEST EIGENVALUE

$A$ (size $n \times n$) has up to $n$ eigenvalue/eigenvector pairs — $n$ different self-consistent "directions." Only one is usable as a centrality score: the one for $\lambda_{max}$.

For a connected graph ($A$ nonnegative and [[Irreducibility|irreducible]]), the Perron-Frobenius theorem guarantees $\lambda_{max}$ is real, simple, and its eigenvector $q_{max}$ has **every entry strictly positive**. Every other eigenvector necessarily mixes signs — a clean proof for symmetric $A$: eigenvectors for distinct eigenvalues are orthogonal, so if $q_{max}$ is all-positive, any other eigenvector $q_k$ must satisfy $q_{max}\cdot q_k = 0$, which is impossible if $q_k$ were also all one sign (the dot product of two same-signed vectors can't be zero). Negative or mixed-sign scores aren't interpretable as "importance," so $q_{max}$ is the only candidate that works.

---

## THE FORMULA, AND WHY IT LOOKS CIRCULAR

$$
EC_n = \frac{1}{\lambda_{max}} A\, q_{max}
$$

This is **not** a from-scratch recipe — $q_{max}$ is an input, already found by solving $Aq_{max} = \lambda_{max}q_{max}$ (e.g. `np.linalg.eig(A)`, taking the eigenvector for the largest eigenvalue). Substituting shows the formula is an identity:

$$
\frac{1}{\lambda_{max}}Aq_{max} = \frac{1}{\lambda_{max}}(\lambda_{max}q_{max}) = q_{max}
$$

It restates the self-consistency property in symbols rather than computing anything new: *take a node's neighbors' scores, sum them, divide by $\lambda_{max}$ — you get that same node's score back.*

---

## THE WALK INTERPRETATION

Repeatedly applying $A$ to a starting vector converges to $q_{max}$ (power iteration) — this is the actual algorithm behind `networkx.eigenvector_centrality`, as opposed to the direct eigendecomposition (`eigenvector_centrality_numpy`). Starting from $v_0 = (1,\dots,1)$, $(A^k v_0)_i$ counts length-$k$ walks touching node $i$ (see the $(A^k)_{ij}$ walk-counting property). So, in the limit, $q_{max,i}$ is proportional to how many long walks pass through node $i$ relative to every other node — the node most "in the flow" of the graph, not just the one with the most direct edges.

---

## WHY THE LARGEST COORDINATE = MOST CENTRAL

From the recursive equation, $x_i$ is large when either:

- $i$ has **many** neighbors contributing to the sum, or
- $i$ has **few** neighbors, but they themselves have large scores.

Both inflate $x_i$ and reinforce each other. This is what separates eigenvector centrality from degree centrality: a degree-1 node attached to a major hub can outrank a degree-3 node attached only to peripheral leaves.

---

## WORKED EXAMPLE

Graph from Cajas, *Advanced Portfolio Optimization*, Ch. 13 ($v_1..v_6$; see the adjacency matrix and walk-counting property under [[Irreducibility]]):

```
eigenvalues: [2.673, -2.093, -1.252, -0.183, 1.206, 0.648]
q_max (normalized): [0.1243, 0.1965, 0.1627, 0.2384, 0.1357, 0.1425]
```

$v_4$ has the largest coordinate. Checking against the graph: $v_4$ has degree 4 (neighbors $v_2, v_3, v_5, v_6$) — the highest in the graph — and one neighbor, $v_2$, is itself well connected (degree 3). $v_4$ wins on both mechanisms above.

Numeric check of the self-consistency identity, row $v_1$ (neighbors $v_2, v_5$):

$$
\frac{q_{max}[v_2] + q_{max}[v_5]}{\lambda_{max}} = \frac{0.1965 + 0.1357}{2.6733} = 0.1243 = q_{max}[v_1]
$$

---

## CODE

```python
import numpy as np

A = np.array([
    [0, 1, 0, 0, 1, 0],
    [1, 0, 1, 1, 0, 0],
    [0, 1, 0, 1, 0, 0],
    [0, 1, 1, 0, 1, 1],
    [1, 0, 0, 1, 0, 0],
    [0, 0, 0, 1, 0, 1],
])

w, v = np.linalg.eig(A)
k = np.argmax(w.real)
lam = w[k].real
q_max = v[:, k].real
q_max = q_max / q_max.sum()  # normalize for readability
```

```python
import networkx as nx

G = nx.from_numpy_array(A)
EC = nx.eigenvector_centrality_numpy(G)
```

---

## RELATED

- [[Linear Algebra]] — index
- [[Irreducibility]] — required for $\lambda_{max}$ to be simple and $q_{max}$ strictly positive; also the source of the $(A^k)_{ij}$ walk-counting property this note leans on
- Perron-Frobenius Theorem — the underlying guarantee for nonnegative irreducible matrices
- PageRank — same recursive-importance idea applied to directed graphs (web links), with damping to handle non-strongly-connected graphs
