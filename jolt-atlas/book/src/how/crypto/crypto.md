# Cryptographic Primitives

Jolt Atlas is parameterized over three cryptographic abstractions. Understanding these traits is the key to understanding how to instantiate or extend the proof system.

---

## JoltField

`JoltField` in `joltworks/src/field/mod.rs` is the trait for finite field arithmetic:

```rust
pub trait JoltField: PrimeField + JoltFieldExt { ... }
```

All polynomial evaluations, sumcheck round messages, and transcript challenges are elements of `JoltField`. The default instantiation is `ark_bn254::Fr`, the BN254 scalar field, a ~254-bit prime field.

BN254 is chosen because:
- Its elliptic curve group supports efficient pairings (needed by HyperKZG).
- It is supported by the Arkworks library with highly optimized arithmetic.
- The field size (~2^254) makes soundness error per sumcheck round negligible.

---

## CommitmentScheme

`CommitmentScheme<Field = F>` in `joltworks/src/poly/commitment/commitment_scheme.rs`:

```rust
pub trait CommitmentScheme: Clone + Send + Sync {
    type Field: JoltField;
    type Setup: Clone;
    type Commitment: Clone + Serialize + ...;
    type Proof: Clone + Serialize + ...;

    fn setup(max_poly_len: usize, rng: &mut impl RngCore) -> Self::Setup;
    fn commit(poly: &MultilinearPolynomial<F>, setup: &Setup) -> (Commitment, Self);
    fn prove(poly: &MultilinearPolynomial<F>, opening_point: &[F], ...) -> Proof;
    fn verify(commitment: &Commitment, opening_point: &[F], value: F, proof: &Proof, ...) -> bool;
}
```

The default instantiation is `HyperKZG<Bn254>`. See [HyperKZG](./hyperkzg.md) for details.

---

## Transcript

`Transcript` in `joltworks/src/transcripts/transcript.rs`:

```rust
pub trait Transcript: Clone + Send + Sync {
    fn new(label: &'static [u8]) -> Self;
    fn append_scalar(&mut self, label: &'static [u8], s: &impl JoltField);
    fn append_serializable(&mut self, label: &'static [u8], t: &impl Serialize);
    fn challenge_scalar<F: JoltField>(&mut self, label: &'static [u8]) -> F;
    fn challenge_vector<F: JoltField>(&mut self, label: &'static [u8], len: usize) -> Vec<F>;
}
```

Transcripts implement the Fiat-Shamir heuristic. Available implementations:

| Type | Hash function | Use case |
|------|-------------|---------|
| `Blake2bTranscript` | BLAKE2b | Default; fast, non-EVM |
| `KeccakTranscript` | Keccak-256 | EVM-compatible verification |

---

## Why Parameterized?

The generic design (`ONNXProof<F, T, PCS>`) means:

1. **Swapping the commitment scheme:** Replace `HyperKZG` with a different PCS (e.g., FRI-based) by implementing `CommitmentScheme` for it.
2. **Swapping the transcript:** Use `KeccakTranscript` for on-chain verification without changing any operator proof code.
3. **Testing:** Use a faster mock PCS for unit tests without affecting production code.

All four operator categories (linear, composite, lookup, multi-stage) are implemented entirely in terms of the `JoltField`, `CommitmentScheme`, and `Transcript` traits. No BN254-specific code leaks into operator implementations.

---

## Default Instantiation

In practice, all examples and the public API default to:

```rust
type DefaultProof = ONNXProof<Fr, Blake2bTranscript, HyperKZG<Bn254>>;
```

The re-exports in `jolt-atlas-core/src/onnx_proof/mod.rs` make this convenient:

```rust
pub use ark_bn254::{Bn254, Fr};
pub use joltworks::{poly::commitment::hyperkzg::HyperKZG, transcripts::Blake2bTranscript};
```
