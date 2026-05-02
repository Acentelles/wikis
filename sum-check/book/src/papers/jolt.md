# Jolt: SNARKs for Virtual Machines via Lookups

| | |
|---|---|
| **Authors** | Arasu Arun, Srinath Setty, Justin Thaler |
| **eprint** | 2023/1217 |
| **Venue** | EUROCRYPT 2024 |

## Summary

Jolt is a zkVM for RISC-V whose front-end produces circuits that perform only lookups, via the **lookup singularity** paradigm. It uses the Lasso lookup argument for large structured tables and the sum-check protocol as its primary PIOP.

## Key Ideas

### Lookup Singularity

Instead of encoding each CPU instruction as a hand-written constraint gadget, Jolt treats every instruction as a lookup into a (virtual) truth table. For a $w$-bit instruction with $k$ input operands, the truth table has $2^{kw}$ entries, which is far too large to materialise. Lasso handles this via **decomposability**: the instruction's truth table decomposes as a combination of smaller sub-tables, each of manageable size.

### Decomposability

An instruction $f : \{0,1\}^{kw} \to \{0,1\}^w$ is $(c, s)$-decomposable if it can be computed from $c$ sub-lookups into tables of size $2^s$ each. All standard RISC-V instructions satisfy this property. The decomposition produces $c$ lookup claims, each proved via Lasso's offline memory checking.

### Lasso

Lasso is a lookup argument for structured tables. It uses offline memory checking (a grand-product argument) to verify that claimed lookup outputs are consistent with the table. The key insight is that the table itself is virtual: its MLE can be evaluated in $O(s)$ time, so the prover never materialises the full table.

### Prover Cost

The prover's per-step cost is dominated by approximately 6 arbitrary 256-bit field-element commitments per RISC-V cycle (compared to $\sim$50 for other approaches like Cairo at the time). The sum-check component is quasi-linear in the trace length.

## Connection to This Book

Jolt is the system whose recursion is addressed by the [Efficient Recursion](./efficient-recursion.md) paper. Its reliance on Dory for polynomial commitments creates the recursion bottleneck that motivates the auxiliary-proof approach. The prover optimisations in [Speeding Up Sum-Check](./speeding-up-sum-check.md) and [Twist and Shout](./twist-and-shout.md) are integrated into the Jolt implementation.
