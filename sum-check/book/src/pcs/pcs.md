# Polynomial Commitment Schemes

A polynomial commitment scheme (PCS) for multilinear polynomials provides four algorithms: Setup, Commit, Open, and Verify. In sum-check-based SNARKs, the PCS is invoked only at the end of the protocol to verify the small number of evaluation claims that sum-check produces.

The PCS choice determines:

- **Proof size**: from $O(1)$ group elements (KZG) to $O(\sqrt{n})$ (Hyrax) to $O(\log^2 n)$ (FRI-based).
- **Verifier time**: often the dominant cost, especially for recursion.
- **Setup**: trusted (KZG, Dory with SRS) vs. transparent (IPA, Hyrax with random generators).
- **Recursion friendliness**: whether the verifier can be efficiently arithmetised.

## Comparison

| PCS | Commitment size | Proof size | Prover | Verifier | Setup | Pairings |
|---|---|---|---|---|---|---|
| Dory | $O(1)$ in $G_T$ | $O(\log n)$ | $O(\sqrt{n})$ multi-pairings | $O(\log n)$ group ops | Transparent | Yes |
| Hyrax | $O(\sqrt{n})$ Pedersen | $O(\sqrt{n})$ | $O(n)$ group ops | $O(\sqrt{n})$ MSM | Transparent | No |
| IPA | $O(1)$ Pedersen | $O(\log n)$ | $O(n)$ group ops | $O(n)$ group ops | Transparent | No |
| HyperKZG | $O(1)$ in $G_1$ | $O(1)$ | $O(n)$ group ops | $O(1)$ pairings | Trusted SRS | Yes |

## Role in Recursion

For recursion, the PCS verifier is the main bottleneck:

- **Dory**: the verifier performs scalar multiplications in $G_1$, $G_2$, and $G_T$ plus a multi-pairing. These are expensive to arithmetise, especially $G_T$ operations (degree-12 extension field). The Jolt recursion paper addresses this by offloading Dory verification to an auxiliary proof.

- **Hyrax**: no pairings, so it can be instantiated over a curve cycle partner (e.g., Grumpkin for BN254). The verifier performs two $O(\sqrt{n})$-sized MSMs, which is the dominant cost in the extended Jolt verifier.

- **HyperKZG**: constant-size proofs with cheap verification (a few pairings). Combined with Spartan, it yields a fast wrapper for on-chain verification at $\sim$5 KB proof size.
