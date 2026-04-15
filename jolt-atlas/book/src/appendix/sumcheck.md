# The Sumcheck Protocol

The sumcheck protocol is the core interactive protocol that underlies every proof in Jolt Atlas. This page gives a self-contained explanation.

---

## The Problem

Given a multilinear polynomial $P: \{0,1\}^k \to \mathbb{F}$, prove to a verifier that:

$$\sum_{x \in \{0,1\}^k} P(x) = C$$

The verifier cannot evaluate $P$ directly (it is too expensive). The prover must convince the verifier efficiently.

---

## The Interactive Protocol

The protocol runs for $k$ rounds. At the start, the prover claims $C_0 = C$.

**Round $j$ (for $j = 1, \ldots, k$):**

1. The prover computes and sends the univariate polynomial:
   $$h_j(X) = \sum_{x_{j+1}, \ldots, x_k \in \{0,1\}^{k-j}} P(r_1, \ldots, r_{j-1}, X, x_{j+1}, \ldots, x_k)$$
   This marginalizes out all variables except the $j$-th. $h_j$ has degree at most $d$ (the total degree of $P$ in variable $j$).

2. The verifier checks: $h_j(0) + h_j(1) = C_{j-1}$. If this fails, abort.

3. The verifier draws a random challenge $r_j \in \mathbb{F}$ and sets $C_j = h_j(r_j)$.

**After round $k$:** The verifier holds a claim $C_k = P(r_1, \ldots, r_k)$. It checks this by evaluating $P$ directly (or via the polynomial commitment opening).

---

## Soundness

By the Schwartz-Zippel lemma, a cheating prover (who sends a wrong polynomial $h_j$) is caught at each round with probability $\ge 1 - d/|\mathbb{F}|$. Over $k$ rounds, the total soundness error is $\le d \cdot k / |\mathbb{F}|$.

For Jolt Atlas with $d \le 4$, $k \le 30$, and $|\mathbb{F}| \approx 2^{254}$:
$$\text{error} \le 4 \cdot 30 / 2^{254} < 2^{-246}$$

This is negligible.

---

## BatchedSumcheck

Jolt Atlas frequently needs to run multiple sumcheck instances over the same hypercube simultaneously (e.g., the three one-hot validity checks). `BatchedSumcheck` combines them with batching coefficients:

1. All $m$ instances append their input claims to the transcript.
2. The verifier draws batching coefficients $\lambda_1, \ldots, \lambda_m$ from the transcript.
3. The combined round message at each round is $\sum_i \lambda_i \cdot h_i^{(j)}(X)$.
4. The combined final claim is $\sum_i \lambda_i \cdot C_i$.

This "front-loaded" batching reduces $m$ sumcheck instances to one, at the cost of one extra field multiplication per element per round.

---

## Gruen's Optimization

Computing the degree-$d$ message $h_j$ at each round naively requires evaluating $P$ at $d+1$ points over the current half-hypercube. Gruen's optimization avoids materializing the full folded polynomial:

- Split the equality polynomial into low and high halves.
- Compute the quadratic and higher cross-terms incrementally.
- Use `par_fold_out_in_unreduced` and `gruen_poly_deg_3` to compute the degree-3 message in $O(2^{k-j}/2)$ field operations with Rayon parallelism.

This is the `compute_message` method in all complex operator provers (Div, Mul, Einsum, etc.).

---

## Non-Interactive via Fiat-Shamir

In Jolt Atlas, all challenges $r_j$ are replaced by hash outputs via the `Transcript`. The prover sends the round messages; the verifier hashes them to get the same challenges. This makes the proof non-interactive and enables the proof to be verified offline.

See [Fiat-Shamir Transcripts](../how/crypto/transcripts.md) for implementation details.
