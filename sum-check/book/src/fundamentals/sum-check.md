# The Sum-Check Protocol

## Statement

Let $g : \mathbb{F}^\ell \to \mathbb{F}$ be an $\ell$-variate polynomial of individual degree at most $d$. The prover claims

$$T = \sum_{b \in \{0,1\}^\ell} g(b)$$

and convinces the verifier of this claim through $\ell$ rounds of interaction.

## Protocol

1. **Round 1.** The prover sends the univariate polynomial $g_1(X_1) = \sum_{b_2,\ldots,b_\ell \in \{0,1\}} g(X_1, b_2, \ldots, b_\ell)$, which has degree at most $d$. The verifier checks $g_1(0) + g_1(1) = T$ and samples a random challenge $r_1 \leftarrow \mathbb{F}$.

2. **Round $j$ ($2 \le j \le \ell$).** The prover sends $g_j(X_j) = \sum_{b_{j+1},\ldots,b_\ell} g(r_1,\ldots,r_{j-1}, X_j, b_{j+1},\ldots,b_\ell)$. The verifier checks $g_j(0) + g_j(1) = g_{j-1}(r_{j-1})$ and samples $r_j \leftarrow \mathbb{F}$.

3. **Final check.** After $\ell$ rounds, the verifier holds a single evaluation claim $g(r_1,\ldots,r_\ell) = g_\ell(r_\ell)$ and checks it against an oracle evaluation of $g$.

## Complexity

| | Operations |
|---|---|
| Prover | $O(d \cdot 2^\ell)$ field ops |
| Verifier | $O(\ell \cdot d)$ field ops |
| Rounds | $\ell$ |
| Soundness error | $\ell d / |\mathbb{F}|$ |

(*Proofs, Arguments, and Zero-Knowledge*, Theorem 4.2)

## Zero-Check Variant

A common application: prove $g(x) = 0$ for all $x \in \{0,1\}^\ell$. The verifier samples $r \leftarrow \mathbb{F}^\ell$ and runs sum-check on

$$0 = \sum_{x \in \{0,1\}^\ell} \widetilde{\text{eq}}(r, x) \cdot g(x).$$

The soundness error increases to $\ell(d+2)/|\mathbb{F}|$ because the integrand has degree $d+1$ per variable. This is the workhorse of Spartan, HyperNova, and the Jolt recursion PIOPs.

## Key Properties

- **Composability.** The final evaluation claim can be verified via another sum-check (producing a "sum-check DAG") or via a polynomial commitment scheme.
- **Flexibility over $H$.** The standard formulation sums over $\{0,1\}^\ell$, but the protocol generalises to any product set $H^\ell \subset \mathbb{F}^\ell$. This flexibility is exploited by the monomial-basis variant (see [Sum-Check over the Monomial Basis](../papers/monomial-basis.md)).
- **Streaming provers.** The prover can be implemented in a streaming fashion with $O(d \cdot 2^{\ell/2})$ memory, or even $O(d \cdot \ell)$ space with additional passes over the input.
