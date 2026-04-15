# Neural Teleport

Neural Teleport is the technique Jolt Atlas uses to prove activation functions with unbounded input domains: **Tanh, Erf, Sigmoid, Sin, Cos**.

---

## The Problem: Unbounded Domain

A standard lookup argument requires a finite table. ReLU is easy: its domain is all integers and its output is determined by the sign bit, which decomposes naturally into 4-bit chunks.

But Tanh has domain $(-\infty, +\infty)$. Materializing a lookup table for all possible `i32` inputs would require $2^{32}$ entries, which is impractical.

---

## The Solution: Reduce to a Bounded Domain

Every periodic function $f$ with period $\tau$ satisfies:
$$f(x) = f(x \bmod \tau)$$

Even for non-periodic functions like Tanh, the function saturates quickly: $\tanh(x) \approx 1$ for $|x| \geq 4$. Jolt Atlas defines a **pseudo-period** $\tau$ large enough to cover the active range, then:

$$x = \tau \cdot q + r, \quad r = x \bmod \tau$$

Only $r \in [0, \tau)$ needs to be looked up in the table. The table has $\tau$ entries, small enough to be practical.

---

## The Teleport Protocol

For an activation function $f$ with pseudo-period $\tau$:

1. **Commit quotient.** The prover commits `TeleportNodeQuotient(node_idx)`, the MLE of $q$ (the quotient $\lfloor x / \tau \rfloor$ for each element).

2. **Commit lookup address.** The prover commits `TeleportRangeCheckRaD(node_idx, d)` for each chunk $d$, the one-hot address for $r = x \bmod \tau$ in the lookup table.

3. **Main sumcheck (`NeuralTeleport`).** Proves:
   $$\text{out}(x) = \text{table}[r(x)]$$
   where $r(x) = x - \tau \cdot q(x)$ and the table value comes from the lookup. The equality $x = \tau \cdot q + r$ is also checked in the same sumcheck.

4. **Range check (`RangeCheck`).** Proves $0 \le r < \tau$ using `UnsignedLessThan` lookup.

5. **One-hot checks (`RaOneHotChecks`).** Proves the address polynomial is a valid one-hot encoding (same three-check batched sumcheck as ReLU).

---

## Lookup Tables

Each activation function has a pre-computed table defined at compile time:

```rust
define_signed_activation_table!(TanhTable, |x: f64| x.tanh(), LOG_TABLE_SIZE);
define_signed_activation_table!(ErfTable,  |x: f64| libm::erf(x), LOG_TABLE_SIZE);
define_signed_activation_table!(SigmoidTable, |x: f64| 1.0 / (1.0 + (-x).exp()), LOG_TABLE_SIZE);
```

The macro takes a closure that computes the function value, evaluates it at each integer input in the table domain, and quantizes the results.

**Scale convention:** The table maps scaled integer inputs to scaled integer outputs. With `SCALE = 128.0`, inputs in $[-128, 128]$ map to outputs in $[-128, 128]$, corresponding to the real range $[-1, 1]$ at 7-bit precision.

---

## Signed Index Conversion

Lookup table indices must be non-negative (they are array indices). Signed `i32` inputs are converted to table indices using two's-complement encoding:

```rust
// Signed i32 → unsigned table index
fn n_bits_to_usize(x: i32, n_bits: usize) -> usize { ... }

// Unsigned table index → signed i32
fn usize_to_n_bits(idx: usize, n_bits: usize) -> i32 { ... }
```

This maps $-128 \to 0$, $-127 \to 1$, $\ldots$, $0 \to 128$, $\ldots$, $127 \to 255$ for an 8-bit table.

---

## Sin and Cos

Sine and cosine are genuinely periodic with period $2\pi$. The teleport technique maps exactly: $\tau = \text{round}(2\pi \cdot 2^s)$ where $s$ is the scale. After reduction modulo $\tau$, the residue $r$ is in $[0, \tau)$, which indexes a small pre-computed table of $\sin(r / 2^s)$ values.

---

## Four ProofType Entries

A neural teleport node generates four entries:

| ProofType | Content |
|-----------|---------|
| `NeuralTeleport` | Main lookup sumcheck |
| `RangeCheck` | Proves $0 \le r < \tau$ |
| `RaOneHotChecks` | Proves one-hot validity of `TeleportRangeCheckRaD` |
| (linked to `TeleportNodeQuotient`) | Eval-reduction links quotient to batch opening |
