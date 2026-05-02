# Sum-Check Is All You Need

| | |
|---|---|
| **Author** | Justin Thaler |
| **eprint** | 2025/2041 |
| **Affiliation** | a16z crypto research, Georgetown University |

## Summary

An opinionated survey arguing that the fastest SNARK provers are those that minimise commitment costs by making maximal use of the sum-check protocol. The paper traces the evolution from interactive proofs through PCPs back to sum-check-based SNARKs, and identifies the key techniques that make modern sum-check provers fast.

## Central Thesis

The fastest known SNARK provers are sum-check-based (Spartan, Lasso, Jolt, HyperNova). They achieve this by:

1. Performing most work in the information-theoretic phase (field arithmetic, quasi-linear cost).
2. Minimising the number and size of polynomial commitments (the expensive cryptographic phase).
3. Using virtual polynomials to avoid committing to derived quantities.

## Key Techniques Surveyed

- **Batch evaluation arguments.** Combining multiple evaluation claims into fewer PCS openings.
- **Read/write memory checking.** Offline memory checking via grand products (the basis for Lasso and Twist/Shout).
- **Virtual polynomials.** Polynomials whose evaluations are checked via sum-check rather than commitment.
- **Sparse sum-checks.** Exploiting sparsity in the polynomial to reduce prover work.
- **Small-value preservation.** Exploiting the fact that committed values are small (e.g., 32-bit) to speed up MSMs.

## Practical Systems Covered

| System | Key innovation |
|---|---|
| vSQL | First practical sum-check SNARK |
| Hyrax | Doubly-efficient, no trusted setup |
| Spartan | Sum-check for R1CS, transparent |
| Lasso | Lookup arguments via memory checking |
| Jolt | zkVM via lookup singularity |
| HyperNova | Recursive arguments for CCS |
