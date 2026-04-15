# Lookup Arguments Background

A lookup argument proves that every element of a vector $f = (f_0, \ldots, f_{N-1})$ appears in a fixed table $T = (T_0, \ldots, T_{M-1})$. This page surveys the landscape and explains why Jolt Atlas uses Shout.

---

## Why Lookups?

Lookup arguments enable efficient proofs for non-arithmetic operations. Rather than expressing $f(x) = \text{ReLU}(x)$ as a polynomial identity (impossible for piecewise functions), the prover instead shows:

> For each index $t$, the value $f_t$ appears at the row $a_t$ of a pre-defined table $T$.

The table is fixed and publicly known. The prover only needs to demonstrate *access correctness*, that the right rows were read.

---

## Plookup / PLONK Lookup

The Plookup argument (GW2020) proves membership via:

1. **Sorted concatenation:** Form the sorted union $s$ of $f$ and $T$.
2. **Grand product check:** Two grand products over $f, T, s$ with a random $\beta$:
   $$\prod_{i} \frac{(1+\beta)(f_i + \beta T_i)}{(s_i + \beta s_{i+1})}$$

**Drawbacks:**
- The prover must sort $s$, which is $O(M \log M)$.
- The grand product requires a permutation argument.
- For large tables (e.g., 64-bit lookup), $M = 2^{64}$ makes direct materialization impossible.

---

## Shout (Read-Once Lookup)

The Shout lookup argument replaces grand products with a **sumcheck over the table entries**:

$$\sum_{j \in [M]} \text{count}(j) \cdot T(j) = \sum_t f_t$$

where $\text{count}(j)$ is the number of times row $j$ was accessed. No sorting, no grand products; just a polynomial evaluation sumcheck.

**Key insight:** For large tables, the table polynomial $T$ is never materialized. Instead, it is represented as a sum of small prefix and suffix sub-tables, each of size $2^c$ for a chunk size $c$. The sumcheck is decomposed over these sub-tables.

Shout is specialized for **read-once** lookups: each table row is accessed at most once. In this case:

- The count polynomial is binary: $\text{count}(j) \in \{0, 1\}$ for all $j$.
- The count polynomial is equivalent to a **one-hot** polynomial over the (address, cycle) product domain.

The one-hot polynomial has only $N$ nonzero entries (where $N$ is the number of lookups), so the HyperKZG commitment costs $O(N)$ rather than $O(M)$.

In Jolt Atlas, most lookups (ReLU, Tanh, range checks) are effectively read-once at the element level, so Shout is the right protocol.

---

## Comparison

| Property | Plookup | Shout |
|----------|---------|-------|
| Prover cost | $O(M + N)$ + sort | $O(N)$ |
| Table materialization | Required | Not needed |
| Grand products | Yes | No |
| Permutation argument | Yes | No |
| One-hot commitment | No | Yes (pay-per-bit) |

---

## Further Reading

- [JOLT paper](https://eprint.iacr.org/2023/1217): Arun, Setty, Thaler (2023)
- [Plookup](https://eprint.iacr.org/2020/315): GW2020 grand product lookup
- [cq](https://eprint.iacr.org/2022/1763): Cached quotient lookup without sorting
