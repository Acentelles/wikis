# Proofs, Arguments, and Zero-Knowledge

| | |
|---|---|
| **Author** | Justin Thaler |
| **Date** | July 2023 (lecture notes, expanded 2015-2023) |
| **Affiliation** | Georgetown University |

## Summary

A comprehensive survey covering five approaches to efficient general-purpose zero-knowledge arguments. This text serves as the canonical reference for the sum-check protocol, the GKR protocol, polynomial IOPs, and polynomial commitment schemes.

## Key Chapters for Sum-Check

- **Chapter 4: Interactive Proofs.** Formal treatment of the sum-check protocol (Theorem 4.2), the GKR protocol for layered arithmetic circuits, and cost analysis.

- **Chapter 5: Fiat-Shamir.** How to remove interaction via hash-based challenge generation.

- **Chapter 6: Front-Ends.** Turning computations into arithmetic circuits or constraint systems (R1CS).

- **Chapters 7-10: From IPs to SNARKs.** The progression from interactive proofs through MIPs, PCPs, and IOPs to practical argument systems.

- **Chapters 15-16: Polynomial Commitments.** Constructions from discrete log (Bulletproofs/IPA, Hyrax, Dory) and pairings (KZG). These chapters define the PCS interface that all sum-check-based SNARKs plug into.

- **Chapter 18: SNARK Composition and Recursion.** How to compose SNARKs and the role of curve cycles.

- **Chapter 19: Bird's Eye View.** Survey of practical argument systems and their tradeoffs, providing context for the more recent papers covered in this book.

## How This Book Uses It

Theorem and definition numbers from this text are cited throughout. The two-component verifier model (information-theoretic + cryptographic) introduced in Chapter 19 is the framing used by both the "Sum-Check Is All You Need" survey and the Jolt recursion paper.
