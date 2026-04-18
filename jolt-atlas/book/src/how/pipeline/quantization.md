# Quantization

Jolt Atlas represents all arithmetic in **fixed-point integers**. Floating-point model weights and activations are quantized to `i32` values before any proof is generated.

---

## Why Integer Arithmetic?

ZK proof systems operate over finite fields, not real numbers. A `MultilinearPolynomial<F>` has evaluations that are field elements (essentially large integers modulo a prime). To bridge the gap between floating-point neural networks and field arithmetic, every tensor value must be an exact integer.

Fixed-point arithmetic achieves this: a real value $v$ is represented as the integer $\text{round}(v \cdot 2^s)$, where $s$ is the scale factor. All arithmetic (add, multiply, divide) can then be performed exactly in `i32`, and the results are correct modulo the quantization error.

---

## The Scale Type

```rust
pub type Scale = i32;
```

A `Tensor<i32>` with scale `s` represents the real-valued tensor $T_\text{real}[i] = T[i] / 2^s$.

Scale is stored as metadata on each tensor:

```rust
tensor.set_scale(8)  // 8-bit fractional precision
tensor.scale()       // read back the scale
```

---

## Quantization Functions

The main entry point is `quantize_tensor` in `atlas-onnx-tracer/src/utils/quantize.rs`:

```rust
pub fn quantize_tensor(tensor: Tensor<f32>, scale: Scale) -> Tensor<i32>;
pub fn quantize_float(value: f32, scale: Scale) -> i32;
```

Conversion: $\text{quantized} = \text{round}(v \cdot 2^s)$

Inverse: $v \approx \text{quantized} / 2^s$

---

## Default Scale

The default quantization scale is 8 bits (configurable per model). This means:

- Values in `[-1, 1]` are represented in the range `[-256, 256]` as `i32`.
- The precision is $1/256 \approx 0.004$ real units per integer step.
- The maximum representable real value is $\text{i32::MAX} / 2^8 \approx 8.4 \times 10^6$.

---

## Arithmetic Scaling Rules

When two tensors are combined arithmetically, their scales must be compatible:

- **Add / Sub:** Both operands must have the same scale. The output has the same scale.
- **Multiply:** If left has scale $s_1$ and right has scale $s_2$, the output has scale $s_1 + s_2$. A right-shift by $s_2$ is applied to keep values in range.
- **Divide:** The quotient scale is $s_1 - s_2$. A left-shift is applied before division to maintain precision.

The `atlas-onnx-tracer/src/utils/precision.rs` module tracks scale propagation through the graph.

---

## Precision Trade-offs

The scale parameter controls a three-way tension between **accuracy**, **overflow risk**, and **proof cost**:

| | Higher scale (e.g. $2^{12}$) | Lower scale (e.g. $2^{7}$) |
|---|---|---|
| **Precision** | Finer granularity ($1/4096$ per step) | Coarser ($1/128$ per step) |
| **Overflow risk** | Intermediate products grow fast; a single matmul with scale $s$ produces values up to $2^{2s} \cdot K$ where $K$ is the contraction dimension | Comfortably within `i32` range |
| **Proof cost** | Larger committed values → more non-zero bits → heavier sumchecks | Smaller polynomials → faster proving |
| **Accuracy** | Near-perfect rank agreement with f32 | Logit direction preserved, but rankings diverge |

### Empirical analysis: GPT-2 at scale $2^7$

The `quant_error_analysis` example runs four views of the same GPT-2 forward pass and compares them:

| View | Arithmetic | Weights | Purpose |
|------|-----------|---------|---------|
| **TRACT** | f32 | Original | Ground-truth reference |
| **QUANT** | i32 fixed-point | Quantized | The actual prover path |
| **SHADOW** | f64 | Quantized | Isolates integer-rounding error from weight-quantization error |

At scale $2^7$ the results show:

- **Cosine similarity** (QUANT vs TRACT): **0.997**, meaning the logit *direction* across the 50,257-dimensional vocabulary is well preserved.
- **Top-1 / Top-5 / Top-10 agreement**: **0%**. Despite high cosine similarity, the absolute logit magnitudes shift enough to completely reorder the ranking. TRACT's top tokens (e.g. " security", " waiter") appear at logits around $-92$, while QUANT's top tokens (e.g. ",", " the") appear around $-3$.
- **Spearman rank correlation**: **0.64**, indicating moderate monotonic agreement across the full vocabulary, but far from rank-preserving in the tail.
- **KL divergence**: **3.2 nats**, reflecting a substantial distributional shift after softmax.

### Where does the error come from?

Comparing SHADOW (f64 arithmetic, quantized weights) against TRACT yields nearly the same error profile: cosine 0.998, top-k agreement 0%, KL 3.6. This means **weight quantization (not integer rounding) is the dominant error source** at this scale. The i32 arithmetic itself introduces negligible additional drift beyond what quantizing the weights already causes.

This has a practical implication: increasing the scale improves accuracy primarily by making weight quantization finer, not by reducing rounding noise in the arithmetic.

### Running the analysis

```bash
cargo run --release --package atlas-onnx-tracer --example quant_error_analysis
```

The example compares QUANT, SHADOW, and TRACT outputs and reports cosine similarity, MSE, KL divergence, top-k agreement, and Spearman/Pearson correlations. It also traces per-node drift to identify which operators accumulate the most error.

---

## Range Constraints

The prover is responsible for ensuring that intermediate `i32` values do not overflow. If any value exceeds `i32::MAX`, the computation produces incorrect results without a detectable error at the proof level. In practice, transformer models with standard weight initialization and 8-bit scale stay well within range.

Division operators require an explicit **range check** to prove that the remainder satisfies $0 \le R < b$. See [Range Checking](../lookups/range-checking.md).

---

## Further Reading

The quantization strategy used in Jolt Atlas draws on a rich body of work in efficient neural network quantization. The papers below are the most relevant references for understanding the design choices and trade-offs involved.

### Foundational Quantization

- **Jacob et al., ["Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference"](https://arxiv.org/abs/1712.05877) (CVPR 2018).**
  The foundational paper on integer-only inference. Introduces the affine quantization scheme (scale + zero-point) used in TensorFlow Lite and establishes the arithmetic scaling rules (e.g., how multiplications change the scale) that Jolt Atlas follows.

- **Gholami et al., ["A Survey of Quantization Methods for Efficient Neural Network Inference"](https://arxiv.org/abs/2103.13630) (2021).**
  Comprehensive survey covering uniform vs. non-uniform quantization, symmetric vs. asymmetric schemes, per-tensor vs. per-channel granularity, and the precision/accuracy trade-off. A good starting point for understanding the quantization landscape.

### Post-Training Quantization for Large Models

- **Dettmers et al., ["LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale"](https://arxiv.org/abs/2208.07339) (NeurIPS 2022).**
  Shows that 8-bit quantization works for billion-parameter transformers with a mixed-precision decomposition for outlier features. Relevant context for why 8-bit fixed-point (the Jolt Atlas default) is sufficient for transformer inference.

- **Frantar et al., ["GPTQ: Accurate Post-Training Quantization for Generative Pre-Trained Transformers"](https://arxiv.org/abs/2210.17323) (ICLR 2023).**
  One-shot weight quantization to 3-4 bits using approximate second-order information. Demonstrates that aggressive weight quantization is feasible for large language models, suggesting future headroom for reducing Jolt Atlas proof sizes.

- **Xiao et al., ["SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models"](https://arxiv.org/abs/2211.10438) (ICML 2023).**
  Migrates quantization difficulty from activations to weights via a mathematically equivalent per-channel scaling transform. Particularly relevant for avoiding activation overflow, a concern shared by ZK integer arithmetic.

- **Lin et al., ["AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration"](https://arxiv.org/abs/2306.00978) (MLSys 2024).**
  Protects salient weight channels (identified by activation magnitude) during quantization. Achieves strong accuracy at low bit-widths without backpropagation, making it practical for post-training quantization pipelines like the one in Jolt Atlas.

### Quantization in zkML

- **Weng et al., ["Mystique: Efficient Conversions for Zero-Knowledge Proofs with Applications to Machine Learning"](https://eprint.iacr.org/2021/730) (USENIX Security 2021).**
  Addresses the cost of converting between fixed-point and field arithmetic inside ZK proofs, directly relevant to Jolt Atlas's quantization-to-field pipeline.

- **Zhang et al., ["A Survey of Zero-Knowledge Proof Based Verifiable Machine Learning"](https://arxiv.org/abs/2502.18535) (2025).**
  Comprehensive survey of zkML systems covering verifiable training, inference, and testing. Reviews how different frameworks handle quantization, precision, and overflow, providing context for where Jolt Atlas sits in the design space.
