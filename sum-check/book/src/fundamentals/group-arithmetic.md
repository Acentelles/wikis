# Group Arithmetic

To prove elliptic curve operations via sum-check, group elements must be represented as multilinear polynomials and their arithmetic encoded as polynomial constraints. This page covers the representation, the PIOPs for each group, and how instances are batched and composed.

All constructions follow the Efficient Recursion paper (Sections 3.1-3.3) and operate inside the auxiliary SNARK, which is native to $\mathbb{F}_q$.

## Representing Group Elements as MLEs

The starting point is the vector-to-MLE identification from the [Multilinear Extensions](./mle.md) page: a vector $v \in \mathbb{F}_q^{2^\ell}$ becomes the unique multilinear polynomial $\tilde{v} : \mathbb{F}_q^\ell \to \mathbb{F}_q$ satisfying $\tilde{v}(b) = v[b]$ for all $b \in \{0,1\}^\ell$.

### $G_T$ elements: 4-variate MLEs

A $G_T$ element lives in $\mathbb{F}_{q^{12}}$, so it is a polynomial $a(X) = a_0 + a_1 X + \cdots + a_{11} X^{11}$ with 12 coefficients in $\mathbb{F}_q$. We need $2^\ell \ge 12$; the smallest choice is $\ell = 4$, giving $2^4 = 16$ evaluation points. The 12 coefficients are laid out as a length-16 vector with 4 zeros of padding:

$$v = (a_0, a_1, \ldots, a_{11}, 0, 0, 0, 0) \in \mathbb{F}_q^{16},$$

and $\tilde{a} : \mathbb{F}_q^4 \to \mathbb{F}_q$ is its MLE. On Boolean inputs, this reads off coefficients:

| $x \in \{0,1\}^4$ (as integer) | $\tilde{a}(x)$ |
|---|---|
| $0, 1, \ldots, 11$ | $a_0, a_1, \ldots, a_{11}$ |
| $12, 13, 14, 15$ | $0, 0, 0, 0$ (padding) |

The padding is inert: it contributes $0 = 0$ to the constraints and does not affect soundness.

### $G_1$ elements: scalar-valued MLEs

$G_1$ points have affine coordinates $(x, y) \in \mathbb{F}_q^2$ plus an infinity indicator $\iota \in \{0,1\}$ (encoded as $\iota = 1, x = y = 0$ for the point at infinity $\mathcal{O}$). Each coordinate is a separate scalar-valued witness MLE indexed over the instance domain $c \in \{0,1\}^k$, where $k = \lceil \log_2 n \rceil$ and $n$ is the number of operation instances.

### $G_2$ elements: split over $\mathbb{F}_{q^2}$

$G_2$ points have coordinates in $\mathbb{F}_{q^2} = \mathbb{F}_q[u]/(u^2+1)$. Each $\mathbb{F}_{q^2}$ element $a = a_0 + a_1 u$ splits into two $\mathbb{F}_q$ components, so a point $(x, y)$ becomes four witness MLEs: $(x_0, x_1, y_0, y_1)$. This doubles the number of constraints compared to $G_1$.

### The general pattern

Take a structured algebraic object (extension-field element, curve point), lay its $\mathbb{F}_q$ components out as a vector of length $2^\ell$, pad to the next power of two, and take the MLE. The sum-check protocol then operates on the MLE as a multilinear polynomial, without needing to know what the underlying object "is."

## $G_T$ PIOPs

### Multiplication

To prove $a \cdot b = c$ in $\mathbb{F}_{q^{12}}$, the [polynomial division identity](./field-arithmetic.md#the-polynomial-division-identity) reduces it to a base-field constraint:

$$C(x) := \tilde{a}(x) \cdot \tilde{b}(x) - \tilde{c}(x) - \tilde{q}(x) \cdot \tilde{p}(x) = 0, \quad \forall x \in \{0,1\}^4.$$

The 4-variate MLEs play a dual role:

1. **On Boolean inputs**, $\tilde{a}(b)$ returns the $b$-th coefficient. The constraint is checked coefficient by coefficient at each of the 16 points.

2. **Off Boolean inputs** (at the random challenge $r \in \mathbb{F}_q^4$), the MLE interpolates smoothly, letting the zero-check reduce 16 point-wise checks to a single evaluation claim.

The polynomial $\tilde{p}$ (MLE of the irreducible's coefficients, padded to 16) is public. The quotient $\tilde{q}$ (11 coefficients, padded to 16) is a prover witness.

Soundness error: $16/|\mathbb{F}_q|$.

### Exponentiation

To prove $h = g^\alpha$ for $g \in G_T$ and $\alpha \in \mathbb{F}_r$, decompose $\alpha$ into base-$b$ digits $(d_0, \ldots, d_{n-1})$ where $n = \lceil \log_b |\mathbb{F}_r| \rceil$, and compute $g^\alpha$ via the recurrence:

$$\rho_{s+1} = \rho_s^b \cdot g^{d_s}, \quad \rho_0 = 1_{G_T}.$$

Let $m = \lceil \log_2 n \rceil$. The prover provides $(m+4)$-variate MLEs:

| Polynomial | Description | Variables |
|---|---|---|
| $\tilde{\rho}(s, x)$ | Accumulator: intermediate values $\rho_s$ | $m + 4$ |
| $\tilde{\rho}_{\text{shift}}(s, x)$ | Shifted accumulator: $\tilde{\rho}(s+1, x)$ | virtual (not committed) |
| $\tilde{q}(s, x)$ | Quotient from the polynomial division identity | $m + 4$ |
| $\tilde{b}(s, x)$ | Digit selector: $\sum_{j=0}^{b-1} \mathbf{1}[d_s = j] \cdot \tilde{g^j}(x)$ | virtual (not committed) |

The step constraint is:

$$C(s, x) := \tilde{\rho}_{\text{shift}}(s, x) - \tilde{\rho}(s, x)^b \cdot \tilde{b}(s, x) - \tilde{q}(s, x) \cdot \tilde{p}(x) = 0, \quad \forall (s, x) \in \{0,1\}^{m+4}.$$

This is verified via a zero-check. After $m + 4$ rounds, the verifier holds evaluation claims on $\tilde{\rho}$, $\tilde{q}$, $\tilde{b}$, and $\tilde{p}$.

**Handling the shift.** The shifted accumulator $\tilde{\rho}_{\text{shift}}$ is not committed. Instead, its evaluation at the challenge point is derived from $\tilde{\rho}$ via a **shift sum-check** using EqPlusOne:

$$\tilde{\rho}_{\text{shift}}(r_s, r_x) = \sum_{(s,x) \in \{0,1\}^{m+4}} \text{EqPlusOne}(r_s, s) \cdot \widetilde{\text{eq}}(r_x, x) \cdot \tilde{\rho}(s, x).$$

This produces a new evaluation claim $\tilde{\rho}(r_3)$ at a fresh point, resolved via oracle access. The verifier then checks boundary conditions: $\tilde{\rho}(0, r_4) = \widetilde{1_{G_T}}(r_4)$ and $\tilde{\rho}(n, r_4) = \tilde{h}(r_4)$.

**Concrete sizing.** For Jolt recursion with $\sigma = 19$ reduce-and-fold rounds and base $b = 4$: $n = \lceil \log_4 |\mathbb{F}_r| \rceil = 127$ steps, $m = \lceil \log_2 127 \rceil = 7$ step variables, giving $m + 4 = 11$ total variables.

Soundness error: $((b+7)(m+4)+8)/|\mathbb{F}_q|$.

## $G_1$ PIOPs

### Point Addition

To prove $R = P + Q$ for $n$ additions indexed by $c \in \{0,1\}^k$, the prover provides 13 $k$-variate witness MLEs:

- **Coordinates and indicators**: $(x_P, y_P, \iota_P)$, $(x_Q, y_Q, \iota_Q)$, $(x_R, y_R, \iota_R)$.
- **Slope**: $\lambda$.
- **Inverse witness**: $\mu := (x_Q - x_P)^{-1}$.
- **Branch indicators**: $\sigma_1$ (doubling), $\sigma_2$ (inverse).

These are encoded in 27 constraints covering:

| Constraints | What they enforce |
|---|---|
| $C_0$-$C_2$ | Booleanity of $\iota_P, \iota_Q, \iota_R$ |
| $C_3$-$C_8$ | Infinity encoding: if $\iota = 1$ then $x = y = 0$ |
| $C_9$-$C_{14}$ | Identity cases: $P = \mathcal{O} \Rightarrow R = Q$ and $Q = \mathcal{O} \Rightarrow R = P$ |
| $C_{15}$-$C_{16}$ | Booleanity of branch selectors $\sigma_1, \sigma_2$ |
| $C_{17}$ | Branch selection: $(1 - \sigma_1 - \sigma_2)(1 - \mu \Delta_x) = 0$ |
| $C_{18}$-$C_{19}$ | Doubling enforcement: $\sigma_1 \cdot \Delta_x = 0$, $\sigma_1 \cdot \Delta_y = 0$ |
| $C_{20}$-$C_{21}$ | Inverse enforcement: $\sigma_2 \cdot \Delta_x = 0$, $\sigma_2 \cdot (y_Q + y_P) = 0$ |
| $C_{22}$ | Slope equation (chord or tangent) |
| $C_{23}$-$C_{26}$ | Output coordinates from slope |

The verifier samples $r_1 \leftarrow \mathbb{F}_q^k$ and $\delta \leftarrow \mathbb{F}_q$, then runs a batched zero-check over the $\delta$-weighted sum $\sum_{j=0}^{26} \delta^j C_j(c)$.

Soundness error: $(7k + 26)/|\mathbb{F}_q|$ (the maximum per-variable degree is $d = 5$, from the branch-selection constraints).

### Scalar Multiplication

To prove $Q = [\alpha]P$ for $P \in G_1$ and $\alpha \in \mathbb{F}_r$, use double-and-add: $A_{i+1} = [2]A_i + b_i P$ starting from $A_0 = \mathcal{O}$, where $(b_0, \ldots, b_{n-1})$ are the bits of $\alpha$ and $n = \lceil \log_2 |\mathbb{F}_r| \rceil$.

Let $m = \lceil \log_2 n \rceil$. The prover provides 8 $m$-variate witness MLEs: accumulator $(x_A, y_A)$, doubled point $(x_T, y_T)$, shifted accumulator $(x'_A, y'_A)$, and indicators $\iota_T, \iota_A$. The base point $(x_P, y_P)$ is constant; the bit MLE $\tilde{b}(i)$ is public. Seven constraints encode denominator-free doubling and conditional addition (branching on $\tilde{b}$ and the infinity indicator).

The shifted accumulator is handled via a shift sum-check, analogous to $G_T$ exponentiation:

$$\tilde{x}'_A(r_s) + \gamma \tilde{y}'_A(r_s) = \sum_{s \in \{0,1\}^m} \text{EqPlusOne}(r_s, s) \cdot \big(\tilde{x}_A(s) + \gamma \tilde{y}_A(s)\big),$$

where $\gamma$ is a batching challenge that combines the two coordinate relations.

Boundary checks: $(x_A, y_A)(0) = \mathcal{O}$ and $(x_A, y_A)(n) = Q$.

Soundness error: $(9m + 9)/|\mathbb{F}_q|$.

## $G_2$ PIOPs

The $G_2$ PIOPs mirror the $G_1$ structure but with coordinates in $\mathbb{F}_{q^2} = \mathbb{F}_q[u]/(u^2+1)$. Each $\mathbb{F}_{q^2}$-valued relation yields two $\mathbb{F}_q$ constraints.

| | $G_1$ | $G_2$ |
|---|---|---|
| Coordinate MLEs per point | 3 $(x, y, \iota)$ | 5 $(x_0, x_1, y_0, y_1, \iota)$ |
| Addition: witness MLEs | 13 | 21 |
| Addition: constraints | 27 | 47 |
| Scalar mul: witness MLEs | 8 | 14 |
| Scalar mul: constraints | 7 | 13 |

Addition soundness: $(7k + 46)/|\mathbb{F}_q|$. Scalar multiplication soundness: $(9m + 17)/|\mathbb{F}_q|$.

The shift sum-check for $G_2$ scalar multiplication batches four $\mathbb{F}_q$-split coordinate MLEs $(x_{A,0}, x_{A,1}, y_{A,0}, y_{A,1})$ with powers of a challenge $\gamma$:

$$\sum_{j=0}^{3} \gamma^j \tilde{w}'_j(r_s) = \sum_{s \in \{0,1\}^m} \text{EqPlusOne}(r_s, s) \cdot \sum_{j=0}^{3} \gamma^j \tilde{w}_j(s).$$

## Batching Instances

In the Dory verifier, the same operation type occurs many times (e.g., up to $10\sigma + 4 = 194$ $G_T$ exponentiation instances for $\sigma = 19$). Instead of running a separate zero-check per instance, all instances of the same type are **stacked** over a constraint-index domain $c \in \{0,1\}^k$ and proved in a single zero-check:

$$\sum_{c \in \{0,1\}^k} \widetilde{\text{eq}}(r_c, c) \cdot \sum_{x \in \{0,1\}^\ell} \widetilde{\text{eq}}(r_x, x) \cdot \tilde{C}(x, c) = 0.$$

This saves both proof size and verifier cost: the verifier receives evaluation claims per operation *type* (exponentiation, multiplication, addition, scalar multiplication), not per instance.

## Composing PIOPs: Copy Constraints

The PIOPs above prove individual operations in isolation. To prove the full Dory verifier, their inputs and outputs must be **wired together**: the output of one $G_T$ exponentiation must equal the input of the subsequent $G_T$ multiplication, and so on through the computation graph.

### Concrete Example: One Reduce-and-Fold Round

In each round of Dory's reduce-and-fold phase, the $G_T$ accumulator updates as:

$$C' = C \cdot \chi_i \cdot D_2^{\beta_i} \cdot D_1^{\beta_i^{-1}} \cdot C_+^{\alpha_i} \cdot C_-^{\alpha_i^{-1}}.$$

This decomposes into separate operations, each proved by its own PIOP instance:

```
[Exp PIOP, instance 1]    D₂, βᵢ    →  D₂^βᵢ       =: t₁
[Exp PIOP, instance 2]    D₁, βᵢ⁻¹  →  D₁^βᵢ⁻¹     =: t₂
[Exp PIOP, instance 3]    C₊, αᵢ    →  C₊^αᵢ       =: t₃
[Exp PIOP, instance 4]    C₋, αᵢ⁻¹  →  C₋^αᵢ⁻¹     =: t₄

[Mul PIOP, instance 1]    C,  χᵢ    →  C·χᵢ        =: m₁
[Mul PIOP, instance 2]    m₁, t₁    →  m₁·t₁       =: m₂
[Mul PIOP, instance 3]    m₂, t₂    →  m₂·t₂       =: m₃
[Mul PIOP, instance 4]    m₃, t₃    →  m₃·t₃       =: m₄
[Mul PIOP, instance 5]    m₄, t₄    →  m₄·t₄       =: C'
```

Each PIOP commits to its own witness polynomials. The exponentiation PIOP for instance 1 has an accumulator $\tilde{\rho}$ whose final value encodes $t_1 = D_2^{\beta_i}$. The multiplication PIOP for instance 2 has an input polynomial $\tilde{b}$ whose value should encode that same $t_1$. The **copy constraint** enforces $\tilde{\rho}(\text{final}) = \tilde{b}(\text{input})$.

### Wiring Edges

Each wiring edge $e$ connects a **source port** (an output polynomial of one PIOP instance) to a **destination port** (an input polynomial of another):

| Edge | Source (operation, port) | Destination (operation, port) |
|---|---|---|
| $e_1$ | Exp instance 1, output $\tilde{\rho}$ | Mul instance 2, input $\tilde{b}$ |
| $e_2$ | Exp instance 2, output $\tilde{\rho}$ | Mul instance 3, input $\tilde{b}$ |
| $e_3$ | Mul instance 1, output $\tilde{c}$ | Mul instance 2, input $\tilde{a}$ |
| $e_4$ | Mul instance 2, output $\tilde{c}$ | Mul instance 3, input $\tilde{a}$ |
| ... | ... | ... |

A port value is not a single scalar but a multilinear polynomial: the $G_T$ wiring ports are $(m'+4)$-variate MLEs encoding 16 $\mathbb{F}_q$ coefficients of a $G_T$ element. So "output equals input" means two polynomials must agree on all Boolean inputs.

### The Wiring Sum-Check

Rather than checking each edge separately, a single sum-check proves all edges at once. The verifier samples random edge-batching challenges $\lambda_e$ and a random evaluation point $r_x$:

$$\sum_{x, c} \widetilde{\text{eq}}(r_x, x) \cdot \sum_{e \in E} \lambda_e \Big(\beta_{\text{src}(e)} \cdot \widetilde{\text{eq}}(c_{\text{src}(e)}, \text{idx}) \cdot V_{\text{src}(e)}(x) - \beta_{\text{dst}(e)} \cdot \widetilde{\text{eq}}(c_{\text{dst}(e)}, \text{idx}) \cdot V_{\text{dst}(e)}(x)\Big) = 0.$$

After the sum-check, the verifier holds evaluation claims on the *already-committed* witness polynomials $V_{\text{src}(e)}$ and $V_{\text{dst}(e)}$ at a shared random point. No additional polynomial commitments are needed, and the edge topology is derived deterministically from the Dory computation DAG.

For $G_1$ points, which have multiple coordinates $(x, y, \iota)$, the port values are batched into a single field element via a transcript challenge $\mu$: $V := x + \mu \cdot y + \mu^2 \cdot \iota$.

Soundness: $(3\ell + 2k + 1)/|\mathbb{F}_q|$.

### Why Not a Permutation Argument?

PLONK-style systems enforce copy constraints via a grand-product accumulator polynomial $Z$ that the prover must commit to. For the Jolt recursion setting, this would be counterproductive:

- The accumulator $Z$ is an $O(n)$-sized polynomial requiring an additional commitment.
- Verifying that additional commitment (via [Hyrax](../pcs/hyrax.md)) costs more than the $O(E)$ "in the clear" cost, because $E \approx 1{,}000\text{-}2{,}000$ is small.
- The sum-check-based wiring batches naturally into the same batched sum-check used for operation constraints (Stage 2 of the PIOP DAG), adding no separate protocol phase.

In short: when the number of wiring edges is small relative to the witness size, paying $O(E)$ directly is cheaper than committing to an $O(n)$-sized accumulator.

## Summary of Witness Costs

For the Dory verifier with $\sigma = 19$:

| PIOP | Committed polys | Vars/poly | Instances |
|---|---|---|---|
| $G_T$ Exp | 2 accumulators $(\tilde{\rho}, \tilde{Q})$ + 5 base-power | $m_{G_T} + 4 = 11$ (accum), $4$ (base-power) | $10\sigma + 4$ |
| $G_T$ Mul | 4 $(\tilde{a}, \tilde{b}, \tilde{c}, \tilde{Q})$ | 4 | $11\sigma + 5$ |
| $G_1$ ScalarMul | 8 step-trace + 2 base-point (constant) | $m_{EC} = 8$ | $3\sigma + 4$ |
| $G_2$ ScalarMul | 14 step-trace + 4 base-point (constant) | $m_{EC} = 8$ | $3\sigma + 4$ |
| $G_1$ Add | 13 | 0 (constant) | $3\sigma + 3$ |
| $G_2$ Add | 21 | 0 (constant) | $3\sigma + 2$ |

After family packing and prefix packing, all of the above collapse into a single 21-variable dense polynomial (~2.1M coefficients).

(*Efficient Recursion for the Jolt zkVM*, Sections 3.1, 4.3, and Appendix A)
