# Fiat-Shamir Transcripts

The Fiat-Shamir transformation converts an interactive proof into a non-interactive one by replacing verifier challenges with hash outputs. Jolt Atlas implements this via the `Transcript` trait in `joltworks/src/transcripts/`.

---

## The Transcript Trait

```rust
pub trait Transcript: Clone + Send + Sync {
    fn new(label: &'static [u8]) -> Self;

    /// Absorb a field element into the transcript state.
    fn append_scalar(&mut self, label: &'static [u8], s: &impl JoltField);

    /// Absorb any serializable value (commitments, field element vectors, etc.).
    fn append_serializable(&mut self, label: &'static [u8], t: &impl Serialize);

    /// Squeeze a field element challenge.
    fn challenge_scalar<F: JoltField>(&mut self, label: &'static [u8]) -> F;

    /// Squeeze a vector of field element challenges.
    fn challenge_vector<F: JoltField>(&mut self, label: &'static [u8], len: usize) -> Vec<F>;
}
```

---

## Available Implementations

### Blake2bTranscript (Default)

Uses BLAKE2b as the underlying hash function. Fast, SNARK-friendly, and not EVM-native.

```rust
use joltworks::transcripts::Blake2bTranscript;
```

Suitable for all proofs that will be verified in a Rust binary.

### KeccakTranscript

Uses Keccak-256 (same as Ethereum's `keccak256`). Slightly slower than BLAKE2b but produces EVM-compatible challenges.

```rust
use joltworks::transcripts::KeccakTranscript;
```

Required for on-chain verification where the verifier smart contract must replay the same hash sequence.

---

## Transcript Ordering Discipline

The ordering of `append` and `challenge` calls is the critical invariant of the Fiat-Shamir transformation. Jolt Atlas follows this strict ordering:

### 1. Witness commitments before any challenge

```
[all PCS::commit() outputs] → transcript.append_serializable(...)
                                    ↓
                            first challenge_scalar() call
```

No challenge is drawn until all polynomial commitments are in the transcript. This prevents the prover from adapting polynomial values based on challenge points.

### 2. Round messages before round challenges

Within each sumcheck instance:
```
append(compressed_unipoly) → challenge_scalar() → ... (repeat k times)
```

### 3. Consistent order across prover and verifier

Both prover and verifier call `append` and `challenge_scalar` in the same order. Since the verifier replays the prover's messages from the proof bytes, the transcript states are identical. This is what makes the non-interactive proof verifiable: all challenges are deterministically derived from all prior messages.

---

## Soundness Under Fiat-Shamir

Under the random oracle model, replacing a random verifier with Fiat-Shamir challenges preserves soundness. A cheating prover would need to find a collision in the hash function (preimage resistance) to control the challenge values after committing to its polynomials.

For BN254 with a 254-bit field, the Schwartz-Zippel soundness error per sumcheck round is $d / |F| < 2^{-246}$ (for degree $d < 2^8$). This is negligible even when multiplied by the number of rounds across all nodes in a GPT-2 proof.
