# Batched Opening and HyperKZG

After the IOP loop, the `ProverOpeningAccumulator` holds all outstanding polynomial opening claims. Rather than opening each polynomial independently, Jolt Atlas reduces all claims to a *single* joint HyperKZG opening via a batching sumcheck.

---

## The Problem Without Batching

The IOP loop generates hundreds of individual polynomial evaluations (one committed polynomial per operator per node, each at its own random evaluation point). A naive proof would require one KZG opening proof per evaluation, making the proof size linear in the number of operators.

Batching reduces this to a constant-size proof regardless of model depth.

---

## Step 1: Batch Opening Sumcheck

`accumulator.prove_batch_opening_sumcheck()` runs a final sumcheck that reduces all individual opening claims to a single joint claim.

The protocol uses random linear combinations to merge all evaluation claims. For claims:
$$P_1(r_1) = v_1, \quad P_2(r_2) = v_2, \quad \ldots, \quad P_m(r_m) = v_m$$

The prover draws batching coefficients $\lambda_1, \ldots, \lambda_m$ from the transcript and proves:
$$\sum_{x \in \{0,1\}^k} \sum_i \lambda_i \cdot \widetilde{eq}(r_i, x) \cdot P_i(x) = \sum_i \lambda_i v_i$$

This is a single sumcheck over the hypercube $\{0,1\}^k$. After $k$ rounds, all claims reduce to a single evaluation point $r^*$ and a single combined claim value $v^*$.

---

## Step 2: HyperKZG Joint Opening

`PCS::prove()` (instantiated as `HyperKZG<Bn254>`) opens all committed polynomials jointly at $r^*$:

```rust
Self::prove_reduced_openings(&mut prover, &poly_map, &pp.generators)
```

This generates a `ReducedOpeningProof`:

```rust
pub struct ReducedOpeningProof<F, T, PCS> {
    pub sumcheck_proof: SumcheckInstanceProof<F, T>,
    pub sumcheck_claims: Vec<F>,
    joint_opening_proof: PCS::Proof,
}
```

The `joint_opening_proof` is a single HyperKZG proof that simultaneously opens all committed polynomials at $r^*$. The BN254 pairing check is performed once.

---

## Why HyperKZG is Efficient Here

HyperKZG operates on multilinear polynomials in evaluation form (as flat `Vec<F>` tables). The Gemini-style reduction maps the multilinear evaluation claim $P(r_1, \ldots, r_k) = v$ to a sequence of univariate KZG claims via a tensor product structure. Because the polynomials are already in evaluation form, no FFT is required, and the prover only needs field operations.

For GPT-2, the HyperKZG step takes approximately 3 seconds, which is a small fraction of the ~38 second total.

---

## Proof Size

The final proof `ONNXProof` has size roughly proportional to:
- $O(n \cdot k)$ field elements for sumcheck round messages ($n$ = number of nodes, $k$ = log of tensor size)
- $O(m)$ group elements for commitments ($m$ = number of committed polynomials)
- $O(\log P)$ group elements for the HyperKZG joint opening ($P$ = total polynomial domain size)

The batched design ensures the HyperKZG proof is $O(\log P)$ regardless of the number of committed polynomials, since all polynomials are opened at the same point.

---

## Verifier Side

The verifier runs:

1. `self.verify_reduced_openings(pp, verifier)`: verifies the batch opening sumcheck and the HyperKZG joint opening using the commitments in `self.commitments`.
2. The pairing check for HyperKZG requires the verifier key (`HyperKZGVerifierKey`), which contains the SRS verification elements (a constant-size public parameter).

Verification time for nanoGPT is ~0.5 seconds.
