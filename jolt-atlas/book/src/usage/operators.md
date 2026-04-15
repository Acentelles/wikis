# Supported ONNX Operators

Jolt Atlas supports 45+ ONNX operators. Each operator has two components:

- **Execution**: implemented in `atlas-onnx-tracer`, runs in integer arithmetic during the forward pass.
- **Proof**: implemented in `jolt-atlas-core`, generates a ZK proof that the execution was correct.

The proof strategy column indicates how the proof is structured:
- **Linear sumcheck**: a single sumcheck instance with no auxiliary committed polynomials.
- **Lookup**: uses the Shout lookup argument backed by pre-committed one-hot polynomials.
- **Composite**: multiple sumcheck instances with pre-committed auxiliary polynomials (quotients, remainders, lookup addresses).

---

## Arithmetic Operators

| Operator | Proof Strategy | Notes |
|----------|---------------|-------|
| `Add` | Linear sumcheck | `output(x) = left(x) + right(x)` |
| `Sub` | Linear sumcheck | `output(x) = left(x) - right(x)` |
| `Mul` | Linear sumcheck (deg 3) | `output(x) = left(x) * right(x)` |
| `Neg` | Linear sumcheck | `output(x) = -input(x)` |
| `Square` | Linear sumcheck (deg 3) | `output(x) = input(x)^2` |
| `Cube` | Linear sumcheck (deg 4) | `output(x) = input(x)^3` |
| `Div` | Composite | Execution sumcheck (deg 3) proving `left = right * q + R`, then Shout range check on remainder + one-hot validity checks |
| `ScalarConstDiv` | Composite | Execution sumcheck (deg 2) proving `operand = divisor * q + R` where divisor is a known constant. Commits remainder polynomial |
| `Rsqrt` | Composite | Execution sumcheck (deg 3) proving two relations batched with γ: inverse (`x * inv + r_i = Q²`) and square root (`sqrt * sqrt + r_s = Q * inv`), then two Shout range checks + one-hot validity checks |

---

## Activation Functions

| Operator | Proof Strategy | Notes |
|----------|---------------|-------|
| `ReLU` | Lookup | Shout lookup against ReLU table + one-hot validity checks |
| `Tanh` | Composite | Neural teleportation: division sumcheck proving `input = quotient * τ + remainder`, then Shout lookup of quotient against a reduced Tanh table, then range check on remainder + one-hot validity checks |
| `Sigmoid` | Composite | Neural teleportation (same structure as Tanh with a Sigmoid table) |
| `Erf` | Composite | Neural teleportation (same structure as Tanh with an Erf table) |
| `Sin` | Composite | Neural teleportation exploiting `2π` periodicity (same structure as Tanh) |
| `Cos` | Composite | Neural teleportation exploiting `2π` periodicity (same structure as Tanh) |
| `Clamp` | Pass-through | Min/max clipping (proof not yet implemented) |

---

## Tensor Operations

| Operator | Proof Strategy | Notes |
|----------|---------------|-------|
| `Einsum` | Linear sumcheck (deg 2) | Sums over contracted dimension using eq-polynomial folding. Covers MatMul and batched variants |
| `Softmax` | Composite | Three batched sumchecks (division by sum, exponent sum, max indicator) + Shout exponentiation lookup + one-hot validity checks |
| `Gather` | Composite | Read-Raf sumcheck (deg 2) for indexed embedding lookup + one-hot validity checks (booleanity, Hamming weight) |
| `Sum` | Linear sumcheck (deg 1) | Summation along a single axis via eq-polynomial folding |
| `Concat` | Linear sumcheck (deg 2) | Batches multiple inputs with per-input eq-selector polynomials |
| `Reshape` | Linear sumcheck (deg 2) | Eq-selector maps input indices to reshaped output indices |
| `Broadcast` | Pass-through | Direct dimension mapping; no sumcheck, just challenge forwarding |
| `MoveAxis` | Pass-through | Permutes challenge variables based on axis reordering; no sumcheck |
| `Slice` | Linear sumcheck (deg 2) | Eq-selector maps output indices back to input indices within slice bounds |
| `Transpose` | Pass-through | Handled via MoveAxis |

---

## Reduction Operators

| Operator | Proof Strategy | Notes |
|----------|---------------|-------|
| `ReduceSum` | Linear sumcheck | |
| `ReduceMean` | Composite | Sum + scalar division |
| `ReduceMax` | Lookup | |
| `ReduceMin` | Lookup | |

---

## Infrastructure Operators

| Operator | Proof Strategy | Notes |
|----------|---------------|-------|
| `Input` | Linear sumcheck | Binds public input into the proof |
| `Constant` | Linear sumcheck | Embeds model constants |
| `Identity` | Linear sumcheck | Pass-through |
| `And` | Lookup | Bitwise AND |
| `Iff` | Linear sumcheck | Conditional select |

---

## Einsum Patterns

The `Einsum` operator covers all matrix multiplication variants used in transformer models:

| Pattern | Description |
|---------|-------------|
| `mk,kn->mn` | Standard matrix-matrix multiply |
| `k,nk->n` | Vector-matrix multiply |
| `bmk,bkn->mbn` | Batched matmul (variant 1) |
| `bmk,kbn->mbn` | Batched matmul (variant 2) |
| `mbk,bnk->bmn` | Batched matmul (variant 3) |
| `mbk,nbk->bmn` | Batched matmul (variant 4) |

The correct variant is selected automatically from the ONNX subscript string via the `EINSUM_REGISTRY` dispatch table.

---

## Operator Coverage Notes

All operators required for standard transformer inference (attention, feedforward, layer norm, embedding) are fully proved. Some less common ONNX operators are implemented in `atlas-onnx-tracer` (execution only) but do not yet have proof implementations in `jolt-atlas-core`. Attempting to prove a model containing an unsupported operator will produce a runtime error identifying the missing operator.
