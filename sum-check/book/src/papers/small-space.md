# Proving CPU Executions in Small Space

| | |
|---|---|
| **Authors** | Vineet Nair, Justin Thaler, Michael Zhu |
| **eprint** | 2025/611 |

## Summary

Shows how to implement Jolt with a significantly reduced memory footprint, without relying on SNARK recursion, and with only modest runtime overhead (potentially well below a factor of two).

## Problem

Existing zkVMs have high prover memory usage. For large traces, the prover must hold the entire witness in memory to compute polynomial commitments and sum-check round messages. This limits the maximum trace length that can be proved on commodity hardware.

## Approach: Streaming Provers

Instead of materialising the full witness, the prover makes multiple streaming passes over the execution trace:

- **Pass 1.** Compute polynomial commitments (MSMs) by streaming over witness coefficients.
- **Pass 2.** Compute sum-check round messages by streaming over the polynomial evaluations.

The key insight is that sum-check round messages for round $j$ depend only on the partial evaluation $g(r_1, \ldots, r_{j-1}, X_j, \cdot)$, which can be computed incrementally.

## Memory Reduction

| Approach | Memory | Runtime overhead |
|---|---|---|
| Standard (materialise all) | $O(n)$ | 1x |
| Streaming (this paper) | $O(\sqrt{n})$ | < 2x |
| Recursive chunking | $O(n/k)$ per chunk | Recursion overhead |

The streaming approach avoids the need for "continuations" (recursive proof aggregation over chunks), which other zkVMs like SP1 and RISC0 rely on to bound memory.

## Comparison with Recursion

Recursion-based chunking introduces:
- Additional prover time for recursive verification.
- Proof composition complexity.
- Potential soundness subtleties from recursive composition.

The streaming approach avoids all of these, at the cost of slightly higher per-step prover time (due to re-reading data). For applications that do not need recursion for other reasons (aggregation, succinct blockchains), streaming is a simpler and potentially faster alternative.

## Connection to Jolt Recursion

The [Efficient Recursion](./efficient-recursion.md) paper notes (footnote 2) that Jolt does not require continuations because its prover memory can be bounded without proof recursion via the streaming techniques from this paper. However, many applications need recursion for other reasons (proof aggregation, incremental verification), which is what the recursion paper addresses.
