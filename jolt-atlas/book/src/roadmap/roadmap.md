# Roadmap

Jolt Atlas is research-grade software under active development. This page describes what is production-ready today and what is planned for the future.

---

## Current Status

The following are implemented and tested end-to-end:

- Proof generation and verification for transformer models (nanoGPT, GPT-2, BGE, Qwen)
- All operators required for transformer inference: Einsum/MatMul, Softmax, ReLU, Tanh, Erf, Sigmoid, Sin, Cos, Gather, Div, Rsqrt, Add, Sub, Mul, Reshape, Concat, Slice, Sum
- HyperKZG polynomial commitment scheme
- Blake2b Fiat-Shamir transcript
- Quantization-aware execution with fixed-point arithmetic

---

## Planned Work

### Input Hiding via BlindFold

**Status:** Design complete (see [Hiding the Inputs with BlindFold](./input-hiding.md)). The BlindFold subprotocol has been ported to `joltworks/` and compiles behind the `zk` feature flag; see [Underway: BlindFold](../underway/blindfold.md) for the current implementation state. Per-operator constraints and ONNXProof integration remain.

Currently `ONNXProof::verify` takes a `&ModelExecutionIO` containing the plaintext input tensors, and the `Input` operator's verifier reconciles a sumcheck input claim by re-evaluating the input MLE in the clear. The plan is a two-step migration: (1) commit input MLEs as `CommittedPolynomial::InputTensor(node_idx)` so they flow through the existing `commit_witness_polynomials` / batch-opening path, and (2) integrate BlindFold to replace cleartext sumcheck round polynomials with Pedersen commitments and cleartext polynomial evaluations with ZK evaluation commitments. Step 1 alone gives the same heuristic guarantee that the [model-hiding](./model-hiding.md) plan uses for hidden weights; Step 2 closes the gap and gives a formal ZK guarantee on the inputs (and on the activation trace as a side benefit, closing the existing "No Zero-Knowledge for Activation Values" caveat).

### Model Hiding

**Status:** Design complete (see `.claude/model-hiding-proposal.md`). Not yet implemented.

Currently, model weights are embedded as public constants in the computation graph. Any verifier with access to the preprocessing key learns the full model. To enable private model inference, the plan is:

1. **`ModelKey` struct**: holds per-layer HyperKZG commitments `{com_ℓ}` to the weight polynomials. Published once; remains fixed for the model's lifetime.
2. **`ModelSecretKey` struct**: holds the actual weight polynomials and blinding randomness `{W̃_ℓ, ρ_ℓ}`. Never leaves the prover.
3. **R1CS restructuring**: weights become witness variables rather than circuit constants. The verifier constructs the constraint system from architecture metadata only (layer sizes, activation types).
4. **Per-inference HyperKZG opening**: at the end of each proof, the prover opens the weight polynomial at the batch challenge point, generating a proof `π_W` of size `O(log P)` (≈ 27 group elements for GPT-2). Prover overhead is a single linear pass over the weights (~few hundred milliseconds).

The hidden-model protocol maintains the same soundness guarantees. The verifier learns only one multilinear evaluation of the weight polynomial at a random point, computationally indistinguishable from uniform under the discrete-log assumption.

### Expanded Operator Coverage

Several ONNX operators are implemented in `atlas-onnx-tracer` (the execution engine) but do not yet have proof implementations in `jolt-atlas-core`. These include some reduction and shape operators used in less common architectures. Contributions are welcome.

### WASM / Browser Verifier

The HyperKZG verifier is relatively lightweight (~0.5 s for nanoGPT). A WASM build would allow in-browser proof verification, enabling client-side zkML applications without a trusted server.

### Recursive Proofs

Wrapping the Jolt Atlas verifier inside a recursive SNARK (e.g., Nova or Supernova) would allow batching multiple inference proofs into a single constant-size proof. This is the standard path toward on-chain verification of large models.

### SRS Precomputation and Caching

The HyperKZG structured reference string (SRS) is currently loaded from disk at startup. For deployment scenarios where key generation time matters, a pre-serialized SRS cache would eliminate this overhead.

---

## Known Limitations

- **No zero-knowledge for activations.** The current prover reveals the computation graph structure (which operators are applied in what order) to the verifier. BlindFold-style ZK for activation values is not yet implemented; the planned integration is described in [Hiding the Inputs with BlindFold](./input-hiding.md), which closes both the activation-leakage caveat and the lack of formal input privacy in a single change.
- **No on-chain verifier.** Verification requires a Rust binary. An EVM-compatible verifier (using `KeccakTranscript`) is a prerequisite for on-chain use.
- **No formal security proof.** The composition of sumcheck instances, lookup arguments, and the batch opening reduction has not been formally analyzed as a complete protocol. Use in production requires an independent security audit.
- **Fixed-point quantization.** The default 8-bit quantization introduces approximation error relative to full-precision inference. The quantization error analysis tool in `atlas-onnx-tracer/examples/quant_error_analysis.rs` can help assess accuracy for a specific model.
