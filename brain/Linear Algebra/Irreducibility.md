---
tags: [linear-algebra, matrices, graph-theory, perron-frobenius]
---
# Irreducibility

A nonnegative square matrix $A$ is irreducible if it cannot be split into independent blocks — informally, every index can reach every other index by "flowing through" nonzero entries. It's the precondition that makes the strong form of the Perron-Frobenius theorem hold, which is why it matters for eigenvector centrality, Markov chains, and any method that ranks or weights nodes by their dominant eigenvector.

---

## MATRIX DEFINITION

$A$ (size $n \times n$, entries $\ge 0$) is **reducible** if there exists a permutation matrix $P$ such that:

$$
P^T A P =
\begin{bmatrix}
B & C \\
0 & D
\end{bmatrix}
$$

with $B$ and $D$ square blocks. This means that, after relabeling indices, one subset of indices never receives flow from another subset — the matrix decomposes into independent (or one-directional) parts.

$A$ is **irreducible** if no such permutation exists (other than the trivial identity).

---

## GRAPH-THEORETIC DEFINITION

Treat $A$ as the adjacency matrix of a (possibly directed) graph: $A_{ij} > 0$ means an edge $v_i \to v_j$.

$A$ is irreducible $\iff$ the graph is **strongly connected** — every node can reach every other node by following edges.

Equivalently, using the walk-counting property of $A^k$ (see [[Linear Algebra]]):

$$
A \text{ is irreducible} \iff \forall i,j\ \exists k \ge 1 \text{ such that } (A^k)_{ij} > 0
$$

i.e. there's a walk of *some* length from every node to every other node. For an **undirected** graph, "strongly connected" collapses to plain **connected**, and $A$ (symmetric) is irreducible exactly when the graph has no isolated components.

---

## WHY IT MATTERS: PERRON-FROBENIUS

For a nonnegative matrix, the Perron-Frobenius theorem gives its strongest guarantees only under irreducibility:

- The largest eigenvalue $\lambda_{max}$ (the Perron root) is real, positive, and **simple** (multiplicity 1 — no other eigenvalue ties it in magnitude).
- Its eigenvector can be chosen with **every entry strictly positive**.

Without irreducibility (a disconnected graph), $\lambda_{max}$ can have multiplicity $> 1$ (one "copy" per disconnected component), and its eigenspace no longer has a single well-defined all-positive direction — a component with no edges to the rest of the graph can end up with a zero entry in the eigenvector, or the top eigenvalue itself can be shared/tied across components, making "the" dominant eigenvector ambiguous. This is precisely the condition eigenvector centrality relies on implicitly when it treats the top eigenvector as *the* ranking vector.

---

## REDUCIBLE EXAMPLE

Two disconnected pairs, $\{v_1, v_2\}$ and $\{v_3, v_4\}$, with no edges between them:

$$
A =
\begin{bmatrix}
0 & 1 & 0 & 0 \\
1 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 \\
0 & 0 & 1 & 0
\end{bmatrix}
$$

This is already block-diagonal — reducible by definition. Each block has its own top eigenvalue of 1, so $\lambda_{max}=1$ has multiplicity 2, and there are two independent nonnegative eigenvectors ($[1,1,0,0]$ and $[0,0,1,1]$), not one — "the" dominant eigenvector isn't unique, so it can't be used as a single ranking.

---

## TESTING IRREDUCIBILITY

Via the walk-counting characterization — check that $I + A + A^2 + \dots + A^{n-1}$ has no zero entries (any pair reachable within $n-1$ steps, if reachable at all):

```python
import numpy as np


def is_irreducible(A: np.ndarray) -> bool:
    n = A.shape[0]
    reach = np.identity(n, dtype=int)
    total = np.identity(n, dtype=int)
    for _ in range(n - 1):
        reach = reach @ A
        total += reach
    return bool(np.all(total > 0))
```

Or, for an undirected graph, simply check connectivity (equivalent, and cheaper):

```python
import networkx as nx

G = nx.from_numpy_array(A)
is_irreducible = nx.is_connected(G)
```

---

## RELATED

- [[Linear Algebra]] — index
- Perron-Frobenius Theorem — eigenvalue/eigenvector guarantees for nonnegative matrices, only fully valid under irreducibility
- Eigenvector Centrality — assumes an irreducible (connected) graph for the dominant eigenvector to be well-defined and all-positive
- Markov Chains — an irreducible Markov chain (every state reachable from every other) is what guarantees a unique stationary distribution, the same underlying theorem applied to stochastic matrices instead of adjacency matrices
