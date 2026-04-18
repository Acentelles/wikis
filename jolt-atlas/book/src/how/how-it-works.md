# How It Works

This section explains the Jolt Atlas proof system from the ground up. It is organized to follow the actual data flow through the code, from an ONNX file to a verified proof.

---

## Sections

### The Proving Pipeline

- **[Architecture Overview](./overview.md)**: The full pipeline in one picture. Read this first.
- **[Phase 1: ONNX to ComputationGraph](./pipeline/onnx-pipeline.md)**: How an ONNX file becomes a typed node graph.
  - [Model Loading and Parsing](./pipeline/model-loading.md)
  - [Quantization](./pipeline/quantization.md)
  - [Tensor Representation](./pipeline/tensors.md)
- **[Phase 2: Forward Pass and Trace](./pipeline/trace.md)**: Executing the model to record intermediate values.
- **[Phase 3: Sumcheck DAG](./pipeline/sumcheck-dag.md)**: How the trace becomes a DAG of sumcheck instances.
  - [Witness Generation and Commitment](./pipeline/witness-commitment.md)
  - [The IOP Loop](./pipeline/iop-loop.md)
  - [Evaluation Reduction](./pipeline/eval-reduction.md)
  - [Batched Opening and HyperKZG](./pipeline/batch-opening.md)

### Operator Proof Strategies

- **[Operator Proof Strategies](./ops/operator-proofs.md)**: The `OperatorProofTrait` interface and operator taxonomy.
  - [Linear Operators](./ops/linear.md): Add, Sub, Mul, Reshape, Concat, …
  - [Division and Rsqrt](./ops/div-rsqrt.md): Euclidean decomposition + range check.
  - [ReLU and Lookup-Based Activations](./ops/relu.md): One-hot lookup arguments.
  - [Neural Teleport](./ops/neural-teleport.md): Tanh, Erf, Sigmoid, Sin, Cos via bounded-domain lookup.
  - [Softmax](./ops/softmax.md): Four-stage composite proof.
  - [Einsum and MatMul](./ops/einsum.md): Tensor contractions as sumchecks.
  - [Gather and Embeddings](./ops/gather.md): Weight table lookups.

### Lookup Arguments

- **[Lookup Arguments](./lookups/lookups.md)**: Why lookups are the central design choice.
  - [The Shout Protocol](./lookups/shout.md): The read-check protocol.
  - [Prefix-Suffix Decomposition](./lookups/prefix-suffix.md): Making large tables tractable.
  - [Range Checking](./lookups/range-checking.md): Proving remainders are in range.

### Cryptographic Primitives

- **[Cryptographic Primitives](./crypto/crypto.md)**: The `JoltField`, `CommitmentScheme`, and `Transcript` traits.
  - [Multilinear Polynomials](./crypto/mlpoly.md): Dense and one-hot polynomial representations.
  - [HyperKZG](./crypto/hyperkzg.md): The polynomial commitment scheme.
  - [Fiat-Shamir Transcripts](./crypto/transcripts.md): Non-interactive proof transformation.

### Appendix

- **[Appendix](../appendix/appendix.md)**: Background reading for readers new to ZK proofs or the specific techniques used.
