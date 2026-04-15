# Security Considerations

This page discusses the security properties of the Jolt Atlas proof system, what is and is not guaranteed, and caveats for production use.

---

## Soundness

### Per-Sumcheck Soundness

Each sumcheck instance has soundness error bounded by the Schwartz-Zippel lemma:

$$\text{error} \le \frac{d \cdot k}{|\mathbb{F}|}$$

where:
- $d$ is the degree of the sumcheck polynomial (at most 4 for the highest-degree operator, Cube)
- $k$ is the number of variables ($\log_2$ of the tensor size, typically $\le 30$)
- $|\mathbb{F}| \approx 2^{254}$ for BN254

This gives $\le 4 \cdot 30 / 2^{254} < 2^{-246}$ per sumcheck instance, which is negligible.

### Composition Soundness

The full proof is a composition of:
- Hundreds of sumcheck instances (one or more per node)
- Evaluation reduction proofs (one per node)
- A batch opening sumcheck
- A HyperKZG joint opening proof

By a union bound, the total soundness error over $n$ sumcheck instances is $n \cdot d \cdot k / |\mathbb{F}|$. For a 300-node model with 3 sumchecks per node, this is $\approx 900 \cdot 120 / 2^{254} < 2^{-237}$, still negligible.

### HyperKZG Security

HyperKZG is secure under the **$q$-Strong Diffie-Hellman** ($q$-SDH) assumption over BN254. This is a standard assumption in the pairing-based cryptography literature.

The SRS must come from a trusted setup ceremony. If the SRS trapdoor $\tau$ is known, a prover can forge arbitrary openings. The pre-computed SRS files in `zkml-jolt-core/` should be replaced with ceremony-generated parameters for production use.

---

## What the Proof Reveals

### Currently Public

| Information | Visible to verifier? |
|------------|---------------------|
| Model architecture (operators, shapes, connectivity) | Yes |
| Model weights (all weight tensors) | Yes |
| Input tensor | Yes |
| Output tensor | Yes |
| Intermediate activations | No (hidden inside sumcheck proofs) |

The verifier has full access to the model; weights are embedded as `Constant` nodes in the preprocessing. This is the **non-hiding** mode.

### With Model Hiding (Planned)

When the model hiding extension is implemented (see [Roadmap](../roadmap/roadmap.md)):

| Information | Visible to verifier? |
|------------|---------------------|
| Model architecture (layer count, sizes, activation types) | Yes |
| Model weights | No (hidden behind HyperKZG commitment) |
| Input tensor | Yes |
| Output tensor | Yes |
| Weight evaluation at one random point | Yes (one field element) |

---

## Fiat-Shamir Security

The non-interactive proof relies on the **random oracle model**: the hash function (BLAKE2b or Keccak) is modeled as a random function. In practice, this is a standard and well-accepted assumption.

The transcript ordering discipline (all commitments before any challenge) is critical. If the prover could see a challenge before committing a polynomial, it could cheat. The code enforces this ordering structurally: `commit_witness_polynomials` runs before `output_claim`, which runs before `iop()`.

---

## Known Limitations

### No Formal Composition Proof

The composition of:
1. Per-operator sumcheck instances
2. Evaluation reduction (line restriction)
3. Batch opening sumcheck
4. HyperKZG joint opening

has not been formally analyzed as a complete protocol in a peer-reviewed paper. Each component is individually well-understood, but the specific composition used in Jolt Atlas has not been independently verified. An independent security audit is recommended before production use.

### No Zero-Knowledge for Activation Values

The current system does not implement BlindFold-style zero-knowledge for the intermediate activation trace. The sumcheck proofs leak information about polynomial evaluations at random points. In practice, this information is computationally useless (random linear combinations of tensor entries), but a formal ZK guarantee is not provided.

### Quantization-Related Soundness

The proof is sound with respect to the **quantized** model, the model that operates in `i32` fixed-point arithmetic. If the quantized model produces different outputs than the full-precision model (which is typical for 8-bit quantization), the proof is correct but proves the wrong statement. The verifier should be aware that the proved computation is the quantized version.

### Trusted Setup Requirement

HyperKZG requires a trusted setup (powers-of-tau ceremony). The SRS files bundled in the repository are generated from a specific random seed and are suitable for development and testing. For production deployments, the SRS should come from a public ceremony with a verifiable transcript.

---

## Recommendations for Production Use

1. **Commission an independent security audit** of the sumcheck composition and batch opening reduction.
2. **Use ceremony-generated SRS** parameters rather than the bundled development SRS.
3. **Validate quantization accuracy** for your specific model using the `quant_error_analysis.rs` tool.
4. **Pin the Rust toolchain** to avoid compiler-version-dependent arithmetic differences.
5. **Do not assume zero-knowledge** for intermediate activations without the BlindFold extension.
