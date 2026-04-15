# Softmax

Softmax is the most complex operator in Jolt Atlas, requiring a four-stage composite proof. It is the dominant non-linear operation in transformer attention mechanisms.

---

## Softmax Definition

For a vector of logits $\mathbf{z}$:

$$\text{softmax}(z_i) = \frac{\exp(z_i)}{\sum_j \exp(z_j)}$$

This requires:
1. Computing $\exp(z_i)$ for each element, a non-linear activation.
2. Summing all exponentials, a reduction.
3. Dividing each exponential by the sum, a division.

In fixed-point integer arithmetic, the sum and max are also needed for numerical stability (log-sum-exp formulation).

---

## Stage 1: Division Sumcheck (SoftmaxDivSumMax)

The first proof combines three things into one sumcheck:

- **Division:** $\text{out}_i = \exp(z_i) / \text{sum}$
- **Sum check:** $\text{sum} = \sum_j \exp(z_j)$  
- **Max check:** $\text{max} = \max_j z_j$

The sum and max are scalar values (the same at every evaluation point), so they can be folded into the division sumcheck with zero extra cost. The combined constraint uses the `SoftmaxRemainder(node_idx, feature_idx)` committed polynomial for the division remainder.

The proof type for this stage is `SoftmaxDivSumMax`.

---

## Stage 2: Exponentiation Lookup (SoftmaxExponentiationReadRaf)

The exponentiation $\exp(z_i)$ is proved via a Neural Teleport-style lookup:

- The prover commits `SoftmaxExponentiationRaD(node_idx, d)`, the one-hot address for each $z_i$ in the exponentiation table.
- A Shout sumcheck proves that each exponentiation output was correctly read from the table.

The proof type is `SoftmaxExponentiationReadRaf`.

---

## Stage 3: One-Hot Validity (SoftmaxExponentiationRaOneHot)

The `SoftmaxExponentiationRaD` committed polynomial must be proved to be a valid one-hot encoding (same three-check batched sumcheck as ReLU).

Proof type: `SoftmaxExponentiationRaOneHot`.

---

## Stage 4: Operand Consistency

The division input (the numerator of softmax, i.e. each exponential) must be the same as the exponentiation output. This is enforced via the identity:

$$\text{softmax\_operand\_claim} = -\text{raf\_claim} + \text{max\_logit}$$

where `raf_claim` is the read-address-function evaluation from the exponentiation sumcheck. This identity links the exponentiation output to the division input without a separate sumcheck.

---

## SoftmaxIndex

The `SoftmaxIndex` struct identifies which feature (attention head / batch dimension) the softmax is being applied to. Transformer attention applies softmax independently across each attention head, so a GPT-2 model has multiple softmax applications per layer.

---

## Committed Polynomials

| CommittedPolynomial | Content |
|--------------------|---------|
| `SoftmaxRemainder(node_idx, feature_idx)` | Division remainder per feature |
| `SoftmaxExponentiationRaD(node_idx, d)` | Exponentiation lookup address, chunk $d$ |

---

## ProofType Summary

| ProofType | Stage |
|-----------|-------|
| `SoftmaxDivSumMax` | Division + sum + max combined |
| `SoftmaxExponentiationReadRaf` | Exponentiation lookup (Shout) |
| `SoftmaxExponentiationRaOneHot` | One-hot validity of exponentiation address |
