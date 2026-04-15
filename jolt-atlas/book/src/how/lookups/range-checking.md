# Range Checking

Range checks prove that a value lies within a specific interval. In Jolt Atlas, they are needed wherever a decomposition produces a remainder that must be bounded to ensure uniqueness of the decomposition.

---

## Why Range Checks Are Needed

The Div execution sumcheck proves:

$$a(x) = b(x) \cdot q(x) + R(x)$$

This constraint is satisfied by any $(q, R)$ pair, not just the unique Euclidean quotient and remainder. A cheating prover could use $R$ outside the range $[0, b)$ to produce a valid-looking execution proof for an incorrect quotient.

The range check closes this gap by additionally proving:

$$0 \le R(x) < b(x) \quad \forall x \in \{0,1\}^k$$

Together, the execution sumcheck and the range check uniquely determine $q$ and $R$.

---

## RangeCheckProvider

`RangeCheckProvider::read_raf_prove` in `jolt-atlas-core/src/onnx_proof/range_checking/mod.rs` generates the range check sumcheck.

The range check uses the `UnsignedLessThan` lookup table in `joltworks/src/lookup_tables/`. This table maps a pair $(R, b)$ (interleaved as a single address) to 1 if $R < b$ and 0 otherwise.

---

## Interleaved Operand Encoding

The range check address polynomial `DivRangeCheckRaD(node_idx, d)` encodes both $R$ and $b$ in a single lookup address by interleaving their bits:

$$\text{address}[t] = \text{interleave\_bits}(R[t], b[t])$$

The `UnsignedLessThan` table is defined over interleaved-bit inputs. This allows a single lookup to prove both the upper bound ($R < b$) and implicitly the lower bound (since $R$ is represented as an unsigned integer, $R \ge 0$ is automatic).

---

## Per-Operator Range Check Polynomials

| Operator | Range check polynomial | Bound |
|----------|----------------------|-------|
| `Div` | `DivRangeCheckRaD(node_idx, d)` | $0 \le R < b$ |
| `Rsqrt` | `RsqrtRangeCheckRaD(node_idx, d)` | inverse in valid range |
| Tanh/Erf/Sigmoid/Sin/Cos | `TeleportRangeCheckRaD(node_idx, d)` | $0 \le r < \tau$ |
| Softmax | `SoftmaxRemainder(node_idx, feature)` | $0 \le R < \text{sum}$ |

---

## Proof Generation

```rust
// In prove_range_and_onehot() for Div:
let range_proof = RangeCheckProvider::read_raf_prove(
    node,
    &prover.trace,
    &prover.poly_map,
    &mut prover.accumulator,
    &mut prover.transcript,
);
proofs.push((ProofId(node.idx, RangeCheck), range_proof));
```

The range check proof is a standard `SumcheckInstanceProof` stored as `ProofType::RangeCheck`.

---

## Verifier Side

The verifier replays the same range check sumcheck from the transcript and checks that the final claim equals the expected value derived from the committed polynomial evaluations. The `UnsignedLessThan` table is publicly known, so the verifier can compute the expected table output for any address.
