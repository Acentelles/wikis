# Multilinear Extensions

## Definition

An $\ell$-variate polynomial $p : \mathbb{F}^\ell \to \mathbb{F}$ is **multilinear** if it has degree at most 1 in each variable. Any such polynomial is uniquely determined by its $2^\ell$ evaluations on the Boolean hypercube $\{0,1\}^\ell$.

Given a function $f : \{0,1\}^\ell \to \mathbb{F}$, the **multilinear extension** (MLE) $\tilde{f} : \mathbb{F}^\ell \to \mathbb{F}$ is the unique multilinear polynomial agreeing with $f$ on all Boolean inputs:

$$\tilde{f}(x_1, \ldots, x_\ell) = \sum_{b \in \{0,1\}^\ell} f(b) \cdot \prod_{i=1}^{\ell} \big(b_i x_i + (1 - b_i)(1 - x_i)\big).$$

(*Proofs, Arguments, and Zero-Knowledge*, Definition 3.5)

## The Equality Polynomial

The equality polynomial $\widetilde{\text{eq}} : \mathbb{F}^{2\ell} \to \mathbb{F}$ is defined as

$$\widetilde{\text{eq}}(x, y) = \prod_{i=1}^{\ell} \big(x_i y_i + (1 - x_i)(1 - y_i)\big).$$

For any $r \in \mathbb{F}^\ell$ and $b \in \{0,1\}^\ell$: $\widetilde{\text{eq}}(r, b) = 1$ if $b = r$ and $0$ otherwise (when $r$ is Boolean). This allows writing evaluation as a sum:

$$\tilde{f}(r) = \sum_{b \in \{0,1\}^\ell} f(b) \cdot \widetilde{\text{eq}}(r, b),$$

which is precisely the form amenable to sum-check.

## EqPlusOne

The shifted equality polynomial $\text{EqPlusOne}(x, y)$ selects pairs where $y = x + 1$ (interpreting the bits as integers, without wraparound). It admits the multilinear expression:

$$\text{EqPlusOne}(r, s) = \sum_{k=1}^{\ell} (1 - r_k) s_k \cdot \prod_{i < k} r_i (1 - s_i) \cdot \prod_{i > k} \big(r_i s_i + (1 - r_i)(1 - s_i)\big).$$

EqPlusOne is computable in $O(\ell)$ via prefix/suffix products and appears in shift sum-checks for proving recurrence relations (e.g., scalar multiplication, exponentiation in the Jolt recursion PIOPs).

## Representation as Vectors

Throughout the sum-check literature, a vector $v \in \mathbb{F}^n$ (where $n = 2^\ell$) is identified with its MLE $\tilde{v} : \mathbb{F}^\ell \to \mathbb{F}$. Committing to $v$ via a polynomial commitment scheme then enables the verifier to check evaluation claims produced by sum-check.
