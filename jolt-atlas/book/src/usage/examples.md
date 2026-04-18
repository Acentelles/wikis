# Running Examples

All end-to-end examples live in `jolt-atlas-core/examples/`. Each example loads an ONNX model, runs the prover, and verifies the resulting proof.

---

## nanoGPT

A ~0.25M-parameter GPT model with 4 transformer layers. The standard benchmark for cross-project comparison. The ONNX model is bundled in the repository, so no download is required.

- **Proving time:** ~16 s (MacBook Pro M3)
- **Parameters:** ~250k (221.2K)

```bash
cargo run --release --package jolt-atlas-core --example nanoGPT
```

---

## GPT-2

OpenAI's GPT-2 with 125M parameters. Requires downloading the model first (see [Quickstart](./quickstart.md) for setup instructions).

- **Proving time:** ~38 s (MacBook Pro M3)
- **Parameters:** 125M

```bash
cargo run --release --package jolt-atlas-core --example gpt2
```

To run text generation without proving:

```bash
cargo run --release --package atlas-onnx-tracer --example gpt2_generate
```

---

## MiniGPT

A smaller GPT variant useful for quick iteration during development.

```bash
cargo run --release --package jolt-atlas-core --example minigpt
```

---

## MicroGPT

A minimal GPT variant for debugging operator implementations. Completes in seconds.

```bash
cargo run --release --package jolt-atlas-core --example microgpt
```

---

## Transformer (Self-Attention Block)

A single self-attention block proof, useful for benchmarking the core transformer primitive in isolation.

```bash
cargo run --release --package jolt-atlas-core --example transformer
```

---

## BGE (Embedding Model)

Proves inference for the BGE small English v1.5 sentence embedding model. Requires downloading the model:

```bash
python scripts/download_bge_small_en_v1_5.py
cargo run --release --package jolt-atlas-core --example bge
```

---

## Qwen

Proves inference for the Qwen language model. Requires downloading the model:

```bash
python scripts/download_qwen.py
cargo run --release --package jolt-atlas-core --example qwen
```

---

## Profiling Flags

All examples accept optional flags after `--`:

| Flag | Effect |
|------|--------|
| `--trace` | Write Chrome Tracing JSON to disk; open with `chrome://tracing` |
| `--trace-terminal` | Print stage timing to terminal |

Example:

```bash
cargo run --release --package jolt-atlas-core --example nanoGPT -- --trace-terminal
```

---

## Tracer-Only Examples

`atlas-onnx-tracer/examples/` contains additional programs that exercise the execution engine without generating proofs:

| Example | Purpose |
|---------|---------|
| `nanoGPT_generate.rs` | nanoGPT forward pass with computation graph summary |
| `gpt2_generate.rs` | GPT-2 text generation (no proof, requires model download) |
| `bge_generate.rs` | BGE embedding model forward pass (requires model download) |
| `inspect_ops.rs` | Print all operators in any ONNX model graph |
| `quant_error_analysis.rs` | Measure quantization error vs. full-precision inference (requires GPT-2 model) |
| `transformer_forward.rs` | Single self-attention block forward pass |
| `multihead_attention.rs` | Multihead attention forward pass |
| `ffn.rs` | Feedforward network (perceptron) forward pass |
