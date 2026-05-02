# Twist and Shout

| | |
|---|---|
| **Authors** | Srinath Setty, Justin Thaler |
| **eprint** | 2025/105 |
| **Affiliation** | Microsoft Research, a16z crypto research, Georgetown University |

## Summary

New memory checking arguments that achieve 10x faster proving than prior methods for logarithmic proof length, and 2-4x even with larger proofs. Two protocols are introduced: **Shout** for read-only memories (lookup arguments) and **Twist** for read-write memories.

## Key Idea: One-Hot Addressing

Instead of encoding memory addresses as integers and checking them via a grand-product permutation argument, Twist and Shout use **one-hot encoding**: each address $a$ is represented by a vector that is 1 at position $a$ and 0 everywhere else. This transforms the memory consistency check into a sum-check over sparse polynomials, where the prover only "pays" for non-zero entries.

## Shout (Read-Only)

For lookup arguments over a table of size $K$:

- The prover commits to a sparse polynomial representing the one-hot-encoded lookups.
- A sum-check proves that the lookup outputs match the table entries.
- For structured memories ($K = T^c$), Shout handles very large tables efficiently.

## Twist (Read-Write)

For read-write memory with $T$ operations:

- Instead of checking a permutation of (address, value, timestamp) tuples, Twist tracks **write increments**: the difference between consecutive writes to each address.
- Higher memory-access locality (e.g., stack operations) leads to sparser polynomials, yielding faster proving.

## Derived SNARKs

- **SpeedySpartan.** A variant of Spartan achieving 4x reduction in commitment costs by using Twist for constraint checking.
- **Spartan++.** An extension handling non-uniform constraints.

## Impact on Jolt

Twist and Shout replace the previous memory-checking arguments in Jolt, reducing the prover's commitment costs for the memory-checking phase. The one-hot addressing approach also interacts well with the small-value optimisation from [Speeding Up Sum-Check](./speeding-up-sum-check.md), since the one-hot vectors have entries in $\{0, 1\}$.
