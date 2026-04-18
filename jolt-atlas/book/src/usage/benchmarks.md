# Benchmarks

**System:** MacBook Pro M3, 16 GB RAM.

**Date:** April 2026.

---

## Model Summary

| Model | Parameters | Prove | Verify | Key gen | Peak RAM |
|-------|-----------|-------|--------|---------|----------|
| MicroGPT | ~1k | 1.05 s | 0.060 s | 0.023 s | 54 MB |
| Transformer (1 block) | ~50k | 4.07 s | 0.175 s | 0.248 s | 374 MB |
| MiniGPT | ~50k | 6.47 s | 0.332 s | 0.064 s | 171 MB |
| nanoGPT (4 layers) | ~250k | **14.6 s** | 0.700 s | 0.209 s | 581 MB |
| GPT-2 | 125M | **~38 s** | — | 0.872 s | — |

---

## nanoGPT vs ezkl

nanoGPT (~0.25M parameters, 4 transformer layers) is the standard cross-project benchmark for zkML systems.

### Jolt Atlas

| Stage | Wall clock |
|-------|-----------|
| Key generation (prover + verifier) | 0.209 s |
| Model trace (forward pass) | 0.372 s |
| Witness commitment | 2.54 s |
| Sum-check proving (IOP) | 7.34 s |
| Reduced openings (batch + HyperKZG) | 4.39 s |
| **Total proof time** | **14.6 s** |
| **Verify time** | **0.700 s** |

### ezkl ([source](https://blog.ezkl.xyz/post/nanogpt/))

| Stage | Wall clock |
|-------|-----------|
| Verifying key generation | 192 s |
| Proving key generation | 212 s |
| Proof time | **237 s** |
| Verify time | 0.34 s |

**Jolt Atlas is ~16x faster on proof generation** (14.6 s vs 237 s). Key generation is also dramatically faster: ezkl requires over 400 seconds of setup versus under 0.25 seconds for Jolt Atlas.

### Why the difference?

ezkl compiles the model to an R1CS/Plonk circuit. Non-linear functions like Softmax and GELU require expensive circuit gadgets. Jolt Atlas proves all operators (including non-linear ones) using sum-check protocols and lookup arguments, which have much lower prover cost.

---

## nanoGPT Stage Breakdown

```
┌──────────────────────────────────┬──────────┬───────┐
│ Stage                            │ Time     │   %   │
├──────────────────────────────────┼──────────┼───────┤
│ Model loading                    │  0.044 s │  0.3% │
│ Key generation                   │  0.209 s │  1.4% │
│ Model trace (forward pass)       │  0.372 s │  2.5% │
│ Witness commitment               │  2.54 s  │ 17.3% │
│ Sum-check proving (IOP)          │  7.34 s  │ 50.1% │
│ Reduced openings (batch + PCS)   │  4.39 s  │ 30.0% │
├──────────────────────────────────┼──────────┼───────┤
│ Total prove                      │ 14.6 s   │       │
│ Total verify                     │  0.700 s │       │
└──────────────────────────────────┴──────────┴───────┘
```

**Observations:**

- **Sum-check proving (50%)** is the dominant cost. This scales with the number of nodes and the degree of each operator's sumcheck polynomial.
- **Reduced openings (30%)** covers the batch opening sumcheck plus HyperKZG joint opening. The large MSM operations at the end (up to 343 ms each) are the main contributors.
- **Witness commitment (17%)** materialises polynomials from the trace and commits via HyperKZG. One-hot polynomials (lookup addresses) benefit from sparse commitment.
- **Key generation (<2%)** is negligible. HyperKZG setup is a single MSM over the SRS.

---

## GPT-2 (125M parameters)

GPT-2 is the largest model benchmarked to date.

| Stage | Wall clock |
|-------|-----------|
| Proving/verifying key generation | 0.872 s |
| Witness generation | ~7.5 s |
| Commitment time | ~3.5 s |
| Sum-check proving | ~16 s |
| Reduction opening proof | ~7 s |
| HyperKZG prove | ~3 s |
| **End-to-end total** | **~38 s** |

**Stage breakdown:**

- **Witness generation (~7.5 s):** The forward pass runs in integer arithmetic. All intermediate tensors are recorded in the `Trace`. Pre-committed auxiliary polynomials (quotients, lookup addresses) are materialized from the trace.
- **Commitment time (~3.5 s):** All auxiliary polynomials are committed via HyperKZG. Commitments are appended to the Fiat-Shamir transcript before any challenge is drawn.
- **Sum-check proving (~16 s):** The IOP loop visits each node in reverse topological order. One or more sumcheck instances are generated per node. This is the dominant cost for transformer-scale models.
- **Reduction opening proof (~7 s):** The evaluation reduction protocol collapses multiple opening requests per polynomial (fan-out) into single claims. A final batching sumcheck reduces all accumulated claims to one joint evaluation point.
- **HyperKZG prove (~3 s):** The Gemini-based HyperKZG prover opens all committed polynomials jointly at the batch challenge point.

---

## Scaling Behaviour

| Model | Nodes | Prove time | Prove/node |
|-------|-------|-----------|------------|
| MicroGPT | ~30 | 1.05 s | ~35 ms |
| Transformer | ~60 | 4.07 s | ~68 ms |
| MiniGPT | ~100 | 6.47 s | ~65 ms |
| nanoGPT | ~250 | 14.6 s | ~58 ms |

Proving time scales roughly linearly with the number of computation nodes, at approximately 50–70 ms per node. The per-node cost depends on operator type: Einsum (matrix multiply) dominates due to its large sumcheck degree, while linear operators (Add, Reshape) are very cheap.

---

## Reproducing the Benchmarks

```bash
# nanoGPT with Chrome flamegraph
cargo run --release --package jolt-atlas-core --example nanoGPT -- --trace
# Then open chrome://tracing and load the generated trace-*.json file

# nanoGPT with terminal timing
cargo run --release --package jolt-atlas-core --example nanoGPT -- --trace-terminal

# Other models
cargo run --release --package jolt-atlas-core --example microgpt -- --trace
cargo run --release --package jolt-atlas-core --example minigpt -- --trace
cargo run --release --package jolt-atlas-core --example transformer -- --trace

# GPT-2 (requires model download first)
python3 scripts/download_gpt2.py
cargo run --release --package jolt-atlas-core --example gpt2 -- --trace
```
