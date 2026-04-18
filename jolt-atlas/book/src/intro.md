# Introduction

**Jolt Atlas** is a zero-knowledge machine learning (zkML) framework that extends the [JOLT](https://github.com/a16z/jolt) proving system to support ML inference verification from ONNX models. Given any ONNX network and an input, Jolt Atlas produces a succinct SNARK proof that a specific output was computed correctly, without requiring the verifier to re-run the network.

Made with ❤️ by [ICME Labs](https://blog.icme.io/).

---

## Why Jolt Atlas?

Traditional zkML systems represent non-linear functions (ReLU, Softmax, Tanh, …) as arithmetic circuits. These circuits are large, expensive to prove, and require complicated auxiliary machinery: quotient polynomials, byte decomposition, grand products, and permutation checks.

Jolt Atlas eliminates all of that. Instead of circuits, every operator (including non-linear activations) is proved using **sum-check protocols** and **lookup arguments** derived from the JOLT framework. The result is a prover that is dramatically faster than circuit-based alternatives.

> **~17× faster** than ezkl on nanoGPT (14 s vs 237 s proof time).
> **GPT-2 (125M parameters)** proved end-to-end in **~38 seconds**.

---

## High-Level Architecture

The proving pipeline has three phases:

```
ONNX file
    │
    ▼  atlas-onnx-tracer
ComputationGraph  ──  node graph with integer arithmetic semantics
    │
    ▼  atlas-onnx-tracer
Trace  ──  all intermediate tensors recorded as i32 values
    │
    ▼  jolt-atlas-core
Witness commitment  ──  auxiliary polynomials committed via HyperKZG
    │
    ▼  jolt-atlas-core
IOP loop (reverse topological)  ──  one or more sumchecks per node
    │
    ▼  jolt-atlas-core
Batch opening  ──  all polynomial openings reduced to one HyperKZG proof
    │
    ▼
ONNXProof  ──  compact, verifiable proof of inference
```

---

## Workspace Crates

| Crate | Role |
|-------|------|
| `atlas-onnx-tracer` | ONNX model loading, quantization, and forward-pass execution |
| `jolt-atlas-core` | SNARK prover and verifier for ONNX computation graphs |
| `joltworks` | Cryptographic primitives: fields, polynomials, sumcheck, HyperKZG |
| `common` | Shared types: `CommittedPolynomial`, `VirtualPolynomial`, constants |

---

## Where to Start

- **New user?** → [Quickstart](./usage/quickstart.md)
- **Running examples?** → [Running Examples](./usage/examples.md)
- **API integration?** → [API Reference](./usage/api.md)
- **Understanding the protocol?** → [Architecture Overview](./how/overview.md)
- **Academic background?** → [Zero-Knowledge Proofs Background](./appendix/zk-background.md)

---

## Related Work

Jolt Atlas builds directly on the [JOLT](https://eprint.iacr.org/2023/1217) paper (Arun et al., 2023). The Shout lookup argument, HyperKZG polynomial commitments, and Gruen-optimized sumcheck are all inherited from the JOLT codebase.
