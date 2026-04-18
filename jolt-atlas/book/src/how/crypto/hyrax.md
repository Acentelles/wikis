# Hyrax

Hyrax is the polynomial commitment scheme used by BlindFold for zero-knowledge openings. Unlike HyperKZG (which handles the main witness commitments), Hyrax commits to polynomials row-by-row using Pedersen vector commitments, enabling a transparent setup and natural integration with sumcheck round commitments.

---

## Matrix Layout

The core idea: instead of committing to all $N$ coefficients of a multilinear polynomial at once, reshape the coefficient vector into an $R \times C$ matrix with $R \cdot C = N$ (typically $R = C = \sqrt{N}$), and commit to each **row** separately with a Pedersen vector commitment.

Let $f$ be a multilinear polynomial in $\ell = \log N$ variables, with coefficients $\{f_{i,j}\}$ indexed by row $i \in [R]$ and column $j \in [C]$. The row commitments are:

$$C_i = \text{Ped}(\text{row}_i,\, \rho_i) = \sum_{j=0}^{C-1} f_{i,j} \cdot G_j + \rho_i \cdot H$$

where $G_0, \ldots, G_{C-1}, H$ are public Pedersen generators and $\rho_i$ is the blinding factor for row $i$. There are $R$ such commitments, each over $C$ bases.

---

## Evaluation Protocol

To evaluate $f$ at a point $r \in \mathbb{F}^\ell$, split $r$ into row and column components: $r = (r_\text{row},\, r_\text{col})$ with $r_\text{row} \in \mathbb{F}^{\log R}$, $r_\text{col} \in \mathbb{F}^{\log C}$.

Using the multilinear Lagrange (eq) basis:

$$f(r) = \sum_{i,j} \widetilde{\text{eq}}(r_\text{row}, i) \cdot \widetilde{\text{eq}}(r_\text{col}, j) \cdot f_{i,j}$$

This factors as:

$$f(r) = \sum_{j=0}^{C-1} \widetilde{\text{eq}}(r_\text{col}, j) \cdot \bar{w}_j$$

where $\bar{w} = \sum_{i=0}^{R-1} \widetilde{\text{eq}}(r_\text{row}, i) \cdot \text{row}_i$ is the **combined row**, a length-$C$ vector that collapses all rows into one via the row-direction Lagrange weights.

---

## Opening Protocol

1. **Prover** computes and sends the combined row $\bar{w}$ (length $C$) and combined blinding $\bar{\rho} = \sum_i \widetilde{\text{eq}}(r_\text{row}, i) \cdot \rho_i$.

2. **Verifier** checks homomorphically that $\bar{w}$ is consistent with the row commitments:

$$\sum_{i=0}^{R-1} \widetilde{\text{eq}}(r_\text{row}, i) \cdot C_i \stackrel{?}{=} \text{Ped}(\bar{w},\, \bar{\rho})$$

This works because Pedersen commitments are linearly homomorphic: the left-hand side is the same linear combination applied to the committed rows.

3. **Verifier** computes the evaluation directly from the revealed combined row:

$$f(r) = \sum_{j=0}^{C-1} \widetilde{\text{eq}}(r_\text{col}, j) \cdot \bar{w}_j$$

---

## Costs

With $R = C = \sqrt{N}$:

| Metric | Cost |
|--------|------|
| Commitment | $N$ group ops total ($\sqrt{N}$ MSMs of size $\sqrt{N}$) |
| Proof size | $\sqrt{N}$ field elements ($\bar{w}$ plus blinding) |
| Verifier work | $\sqrt{N}$ group ops (homomorphic check) + $\sqrt{N}$ field ops (final dot product) |
| Setup | Transparent: $C + 1$ Pedersen generators, no trusted setup |

Compared to KZG ($\log N$ proof size, constant verifier pairings), Hyrax trades proof succinctness for transparency and hiding.

---

## Why Hyrax in BlindFold

BlindFold uses Hyrax specifically for the ZK opening layer, not as Jolt Atlas's main PCS. The reasons are structural:

### 1. Free reuse of sumcheck round commitments

During the ZK sumcheck (Phase 1), the prover already produces one Pedersen commitment per round to the round polynomial's coefficients. BlindFold arranges the witness so that these round commitments **are exactly the row commitments** of the Hyrax grid. The "commit" step for openings is already paid for; only the combined-row message and the homomorphic check remain.

### 2. Homomorphic folding under Nova

Row commitments fold under Nova's folding scheme. When two R1CS instances are folded with challenge $r$, the row commitments combine as $C'_i = C_{\text{real},i} + r \cdot C_{\text{rand},i}$. The Hyrax opening protocol carries through the folded instance without structural changes, because the linear homomorphism of Pedersen commitments is preserved.

### 3. Transparent, discrete-log security

Hyrax requires only Pedersen generators (no trusted setup, no SRS). This aligns with BlindFold's security model: the blinding factors $\rho_i$ make each row commitment perfectly hiding, and opening security relies on the discrete logarithm assumption.

### 4. Natural grid alignment

The Hyrax grid dimensions ($R' \times C$ for the witness, $R_E \times C_E$ for the error vector) match how BlindFold's R1CS witness is organized. The coefficient-row commitments from Phase 1 map directly to rows in the grid, so no reshaping or re-commitment is needed.

---

## Implementation in Jolt Atlas

The Hyrax opening logic lives in `joltworks/src/poly/commitment/pedersen.rs` under the `pedersen::hyrax` submodule:

| Function | Purpose |
|----------|---------|
| `combined_row` | Computes $\bar{w} = \sum_i \widetilde{\text{eq}}(r_\text{row}, i) \cdot \text{row}_i$ |
| `combined_blinding` | Computes $\bar{\rho} = \sum_i \widetilde{\text{eq}}(r_\text{row}, i) \cdot \rho_i$ |
| `evaluate` | Computes $f(r) = \sum_j \widetilde{\text{eq}}(r_\text{col}, j) \cdot \bar{w}_j$ |
| `HyraxOpeningProof` | Bundles $\bar{w}$ and $\bar{\rho}$ for serialization |

The `protocol.rs` orchestrator calls these helpers in the final phase of BlindFold to open $W(r_y)$ and $E(r_x)$ against the folded row commitments. See [BlindFold](../../underway/blindfold.md) for the full protocol flow.

---

## Reference

Wahby, Tzialla, shelat, Thaler, Walfish. *Doubly-Efficient zkSNARKs Without Trusted Setup.* [eprint 2017/1132](https://eprint.iacr.org/2017/1132.pdf), 2018.
