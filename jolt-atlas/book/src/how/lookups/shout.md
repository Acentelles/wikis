# The Shout Protocol

Jolt Atlas uses the Shout lookup argument, a read-once lookup protocol, to prove that non-linear operator outputs were correctly read from their lookup tables.

---

## The Lookup Problem

Given:
- A fixed lookup table $T$ of size $M = 2^{\log M}$
- A sequence of $N$ addresses $a_0, a_1, \ldots, a_{N-1} \in [0, M)$
- A sequence of $N$ values $v_0, v_1, \ldots, v_{N-1}$

Prove: $v_t = T[a_t]$ for all $t$.

---

## Read-Address-Function (RAF) Protocol

The Shout protocol is implemented as a read-raf sumcheck. The key insight is that the address polynomial `addr_poly` can be represented as a **one-hot** polynomial; for each cycle $t$, exactly one table row is selected:

$$\text{addr\_poly}(j, t) = \mathbf{1}[j = a_t]$$

This sparse representation allows $O(N)$ commitment cost rather than $O(M)$.

The `ReadRafSumcheckParams` struct in `joltworks/src/subprotocols/` encodes this structure for the sumcheck prover.

---

## Two Phases

The sumcheck has two phases that bind different sets of variables:

### Address Phase

Binds the $\log M$ address variables. In this phase, the address polynomial is evaluated by the prefix-suffix decomposition (see [Prefix-Suffix Decomposition](./prefix-suffix.md)). The prover uses cached prefix evaluations to avoid recomputing from scratch each round.

### Cycle Phase

Binds the $\log N$ cycle variables. In this phase, the equality polynomial $\widetilde{eq}(r_t, \cdot)$ restricts the sum to the output evaluation point.

---

## Gamma Batching

The full lookup argument combines the read-raf check with the operator's output constraint via a batching coefficient $\gamma$ drawn from the transcript:

$$\text{combined\_claim} = \gamma \cdot \text{raf\_claim} + \text{output\_claim}$$

This merges the lookup correctness proof with the execution correctness proof into a single sumcheck, reducing proof overhead.

---

## Three One-Hot Validity Checks

After the main Shout sumcheck, three additional sumchecks prove the address polynomial is a valid one-hot encoding:

1. **Booleanity:** $\text{addr}(j, t) \cdot (1 - \text{addr}(j, t)) = 0$ for all $j, t$. Ensures entries are 0 or 1.

2. **Hamming weight:** $\sum_j \text{addr}(j, t) = 1$ for each cycle $t$. Ensures exactly one entry per cycle is 1.

3. **Hamming booleanity:** $\text{addr}(j, t) \cdot \text{addr}(j', t) = 0$ for $j \neq j'$. Ensures at most one entry per cycle is 1.

These three checks are combined via `BatchedSumcheck` with batching coefficients $\lambda_1, \lambda_2, \lambda_3$ from the transcript.

---

## Code Locations

| Component | File |
|-----------|------|
| Shout sumcheck prover | `joltworks/src/subprotocols/` |
| RAF virtual polynomial | `joltworks/src/subprotocols/ra_virtual.rs` |
| Batched sumcheck | `joltworks/src/subprotocols/sumcheck.rs` (BatchedSumcheck) |
| Op lookup provider | `jolt-atlas-core/src/onnx_proof/op_lookups/mod.rs` |
| Range check provider | `jolt-atlas-core/src/onnx_proof/range_checking/mod.rs` |
