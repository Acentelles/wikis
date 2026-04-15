# Quickstart

This guide walks you from a fresh clone to a running end-to-end proof.

## Prerequisites

- **Rust** (nightly). The repository includes a `rust-toolchain.toml` that pins the exact nightly version; Cargo will install it automatically on first build.
- **Git** and standard build tools (`gcc` / `clang`, `make`).

No Python is needed unless you want to run the GPT-2 example (which requires downloading the model from Hugging Face).

## Step 1: Clone the Repository

```bash
git clone https://github.com/ICME-Lab/jolt-atlas.git
cd jolt-atlas
```

## Step 2: Run the nanoGPT Example

nanoGPT is a ~0.25M-parameter GPT model bundled directly in the repository (no download required).

```bash
cargo run --release --package jolt-atlas-core --example nanoGPT
```

The first build compiles all four workspace crates and will take several minutes. Subsequent builds are incremental.

**Expected output:**

```
=== nanoGPT ===
(Run with --trace for JSON output or --trace-terminal for terminal output)
```

The example loads the model, generates and verifies a proof, then exits silently on success. Timing will vary by machine; on a MacBook Pro M3 the proof takes approximately 16 seconds.

## Step 3: Interpret the Output

Add `-- --trace-terminal` to print a timing breakdown to the terminal:

```bash
cargo run --release --package jolt-atlas-core --example nanoGPT -- --trace-terminal
```

Add `-- --trace` to generate a Chrome Tracing JSON file:

```bash
cargo run --release --package jolt-atlas-core --example nanoGPT -- --trace
```

Open `chrome://tracing` in Chrome and load the generated file to see a flamegraph of the proof stages.

## Step 4: Run GPT-2 (Optional)

GPT-2 (125M parameters) requires downloading the ONNX model from Hugging Face. A helper script handles this automatically.

```bash
# Create a Python virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Download and prepare the model
python scripts/download_gpt2.py
```

This exports GPT-2 via [Hugging Face Optimum](https://huggingface.co/docs/optimum/index) into `atlas-onnx-tracer/models/gpt2/`.

Test the model loads correctly (no proof, just execution):

```bash
cargo run --release --package atlas-onnx-tracer --example gpt2_generate
```

Then prove and verify:

```bash
cargo run --release --package jolt-atlas-core --example gpt2
```

A successful run exits silently after approximately 38 seconds on an M3 MacBook Pro. Add `-- --trace-terminal` for a detailed timing breakdown.

## What's Next?

- See all available examples: [Running Examples](./examples.md)
- Integrate the API into your own code: [API Reference](./api.md)
- Understand how the proof system works: [Architecture Overview](../how/overview.md)
