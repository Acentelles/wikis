# Zero-Knowledge Proofs Background

This page introduces zero-knowledge proofs for readers new to the field.

---

## What is a Zero-Knowledge Proof?

A zero-knowledge proof (ZKP) is a protocol between a **prover** and a **verifier** where:

- The **prover** knows some secret information (a *witness*) and wants to convince the verifier that a statement is true.
- The **verifier** wants to be convinced, but does not want to learn the prover's secret.

Formally, a ZKP must satisfy three properties:

1. **Completeness:** An honest prover can always convince an honest verifier of a true statement.
2. **Soundness:** A cheating prover cannot convince a verifier of a false statement, except with negligible probability.
3. **Zero-knowledge:** The verifier learns nothing beyond the truth of the statement. More precisely, the verifier's view can be *simulated* without knowing the witness.

---

## SNARKs

A **SNARK** (Succinct Non-interactive ARgument of Knowledge) is a special kind of proof with two additional properties:

- **Succinct:** The proof is short (much shorter than the computation being verified) and fast to verify.
- **Non-interactive:** The prover sends a single message; no back-and-forth communication is needed.

SNARKs are constructed from interactive proofs (like the sumcheck protocol) via the Fiat-Shamir transformation, which replaces verifier challenges with hash outputs.

---

## Why SNARKs for ML Inference?

In a typical ML deployment:

1. A model owner has a proprietary model.
2. A user queries the model with an input.
3. The user wants to trust that the returned output is correct, that the model owner didn't fake the result.

Without ZK, the user must either re-run the model (expensive) or trust the model owner (centralized). With a SNARK:

- The model owner generates a proof that the output was produced by running the model on the input.
- The user verifies the proof in seconds, much faster than re-running the model.
- If the model is private, zero-knowledge properties can hide the model weights while still proving inference correctness.

---

## The JOLT Connection

Jolt Atlas extends the [JOLT](https://eprint.iacr.org/2023/1217) proving system. JOLT introduced:

- **Shout:** A lookup argument based on decomposed table sums rather than grand products, optimized for read-once lookups.
- **JOLT zkVM:** An application of Shout to prove RISC-V instruction execution.

Jolt Atlas applies the same techniques to ONNX model inference: instead of proving CPU instruction execution, it proves neural network operator execution.

---

## Further Reading

- [JOLT paper](https://eprint.iacr.org/2023/1217): Arun, Setty, Thaler (2023)
- [Groth16](https://eprint.iacr.org/2016/260): Classical pairing-based SNARK
- [PLONK](https://eprint.iacr.org/2019/953): Universal SNARK with lookup arguments
- [Spartan](https://eprint.iacr.org/2019/550): Transparent SNARK using sumcheck
- [HyperPlonk](https://eprint.iacr.org/2022/1355): Sumcheck-based PLONK
- [Gemini](https://eprint.iacr.org/2022/420): Multilinear KZG
