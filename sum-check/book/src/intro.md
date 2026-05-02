# Introduction

This book surveys the sum-check protocol and the ecosystem of techniques built around it for constructing SNARKs with fast provers. The sum-check protocol, introduced by Lund, Fortnow, Karloff, and Nisan in 1992, has become the backbone of the most efficient known SNARK provers.

The central observation is that the fastest SNARK provers minimize the use of expensive cryptographic operations (multi-scalar multiplications, pairings) by delegating as much work as possible to information-theoretic protocols, primarily sum-check. The cryptographic cost is then confined to a polynomial commitment scheme (PCS), which is invoked only on a small number of evaluation claims produced by the sum-check.

## Scope

The book is organised in three parts:

1. **Fundamentals.** The sum-check protocol itself, multilinear extensions, virtual polynomials, and the path from interactive proofs to deployed SNARKs.

2. **Polynomial commitment schemes.** The PCS layer that plugs into sum-check-based PIOPs: Dory, Hyrax, IPA/Bulletproofs, and HyperKZG.

3. **Papers.** Per-paper entries covering the key results, techniques, and open problems in this area.

## Canonical reference

Justin Thaler's *Proofs, Arguments, and Zero-Knowledge* (2023) serves as the canonical reference throughout. Chapter and theorem numbers refer to that text unless stated otherwise.
