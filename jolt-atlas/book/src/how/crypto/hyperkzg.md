# HyperKZG

HyperKZG is the polynomial commitment scheme used by Jolt Atlas. It extends the classical KZG scheme to multilinear polynomials via the Gemini-style tensor product transformation.

---

## KZG Background

The classical KZG scheme (Kate-Zaverucha-Goldberg) commits to a univariate polynomial $p(X)$ of degree $d$ as:

$$\text{com}(p) = [p(\tau)]_1 = \sum_{i=0}^{d} p_i [\tau^i]_1$$

where $\tau$ is a secret element from the structured reference string (SRS). To open $p(z) = v$, the prover produces a KZG proof:

$$\pi = \left[\frac{p(X) - v}{X - z}\right]_1$$

The verifier checks $e(\text{com} - [v]_1, [1]_2) = e(\pi, [\tau - z]_2)$ using the BN254 pairing.

---

## From Univariate to Multilinear: Gemini

HyperKZG uses the Gemini transformation to reduce a multilinear evaluation claim $P(r_1, \ldots, r_k) = v$ to a sequence of univariate KZG claims.

The key observation: a multilinear polynomial in $k$ variables with evaluation table $(v_0, \ldots, v_{2^k - 1})$ can be written as a tensor product. The Gemini protocol exploits this structure to fold the $k$-dimensional evaluation into a chain of $k$ univariate evaluations, one per variable.

Because Jolt Atlas represents polynomials in **evaluation form** (flat `Vec<F>`) rather than coefficient form, no FFT or interpolation is needed. The prover only performs field multiplications, with no polynomial arithmetic.

---

## Setup and Keys

```rust
// Setup (done once per model; loads from disk if pre-generated)
let srs = HyperKZGSRS::setup(&mut rng, max_poly_len);
let (prover_key, verifier_key) = srs.trim(poly_len);
```

The SRS consists of group elements $[\tau^i]_1$ and $[\tau^i]_2$ for $i = 0, \ldots, N$. Pre-computed SRS files for large polynomial sizes are stored in `zkml-jolt-core/`:

- `dory_srs_28_variables.srs`: supports up to $2^{28}$ elements
- `dory_srs_29_variables.srs`: supports up to $2^{29}$ elements

```rust
// Load from file (faster than re-generating)
let srs = HyperKZGSRS::load_from_file("path/to/srs.srs");
```

---

## Prover Key and Verifier Key

```rust
pub struct HyperKZGProverKey<C: Pairing> {
    kzg_pk: KZGProverKey<C>,
    // Generators [τ^0]_1, [τ^1]_1, ... for committing polynomials
}

pub struct HyperKZGVerifierKey<C: Pairing> {
    kzg_vk: KZGVerifierKey<C>,
    // [τ]_2 for pairing checks; [1]_1 for evaluation checks
}
```

The verifier key is small (a constant number of group elements) and can be distributed publicly.

---

## Commitment

```rust
let (commitment, prover_key) = HyperKZG::<Bn254>::commit(&poly, &srs);
```

For a dense polynomial of $2^k$ field elements, this is a multi-scalar multiplication (MSM) of size $2^k$. Jolt Atlas uses the `msm/` module in `joltworks` with optimized MSM algorithms.

For a one-hot polynomial with $N$ nonzero entries, the MSM has size $N$, the pay-per-bit property.

---

## Opening and Verification

```rust
// Prover: open at a single point r ∈ F^k
let proof = HyperKZG::<Bn254>::prove(&poly, &[r], &prover_key, transcript);

// Verifier: check the opening
let valid = HyperKZG::<Bn254>::verify(&commitment, &[r], value, &proof, &verifier_key, transcript);
```

The proof consists of $k$ group elements (one per variable) plus a final KZG pairing check.

---

## Joint Opening (Batch Mode)

In the batch opening step, all committed polynomials are opened at the same point $r^*$ (the output of the batch opening sumcheck). This allows a single joint KZG proof:

```rust
HyperKZG::batch_prove(&polys, &r_star, &values, &prover_key, transcript)
```

The batch proof is $O(\log P)$ group elements regardless of the number of polynomials, since the Gemini transform handles all of them with a shared SRS evaluation.
