# Prefix Packing

At the end of a sum-check-based PIOP, the verifier holds $c$ evaluation claims on committed multilinear polynomials of varying sizes. Each claim must be opened against its commitment via a polynomial commitment scheme. Prefix packing reduces all $c$ claims to a **single dense polynomial opening**, avoiding both per-polynomial commitment overhead and wasteful padding.

## The Problem

Suppose the PIOP produces evaluation claims on polynomials $\tilde{f}^{(1)}, \ldots, \tilde{f}^{(c)}$, where $\tilde{f}^{(i)}$ has $n_i$ variables (and hence $2^{n_i}$ coefficients). There are two naive approaches, both costly:

1. **Separate commitments.** Commit to each polynomial individually. For [Hyrax](../pcs/hyrax.md) over Grumpkin, each commitment is a vector of $O(\sqrt{2^{n_i}})$ Pedersen commitments. The total cost is linear in $c$, and the verifier must check $c$ separate openings.

2. **Pad to uniform size.** Pad all polynomials to $n_{\max} = \max_i n_i$ variables and commit to a single polynomial. The prover's cost becomes proportional to $c \cdot 2^{n_{\max}}$ instead of $\sum_i 2^{n_i}$, which can be much larger when the polynomials have different sizes.

Prefix packing avoids both problems.

## The Construction

### Assigning Prefix Codes

Given $c$ multilinear polynomials $\tilde{f}^{(1)}, \ldots, \tilde{f}^{(c)}$ with $\tilde{f}^{(i)}$ on $n_i$ variables, construct a single packed multilinear polynomial $\tilde{f}$ on $n = \lceil \log_2(\sum_i 2^{n_i}) \rceil$ variables.

Each polynomial is assigned a unique binary prefix $b^{(i)} \in \{0,1\}^{n - n_i}$ forming a **prefix-free code** (no prefix is a prefix of another). The packed polynomial is then:

$$\tilde{f}(z) = \begin{cases} \tilde{f}^{(i)}(x) & \text{if } z = (b^{(i)} \| x) \text{ for some } i, \\ 0 & \text{otherwise.} \end{cases}$$

Each polynomial $\tilde{f}^{(i)}$ occupies a disjoint subcube $\{(b^{(i)}, x) \mid x \in \{0,1\}^{n_i}\}$ of the Boolean hypercube $\{0,1\}^n$.

### Why Prefix Codes Exist

The required prefix lengths are $l_i = n - n_i$. By Kraft's inequality:

$$\sum_i 2^{-l_i} = 2^{-n} \sum_i 2^{n_i} \le 1$$

by choice of $n$. A simple greedy algorithm assigns prefixes: sort polynomials by $n_i$ descending, maintain a counter $v = 0$, write $v$ in binary, right-pad to length $l_i$ to get $b^{(i)}$, then set $v \leftarrow \text{integer}(b^{(i)}) + 1$. Right-padding ensures power-of-two alignment; monotonicity of $v$ ensures no code is a prefix of another.

### Overhead

The packed size satisfies $2^n \le 2 \sum_i 2^{n_i}$, so the packing introduces **at most 2x overhead** compared to the total size of the individual polynomials.

## Reducing $c$ Claims to One Opening

After the batched sum-check reduces all claims to a shared evaluation point $r_x$, the verifier holds virtual openings $\tilde{f}^{(i)}(r_x) = v_i$ for each polynomial. To verify all of them with a single PCS opening:

1. Sample fresh packing challenges $r_{\text{pack}}$ from the transcript.

2. Compute the **subcube selector** for each prefix:

$$w_i(r_{\text{pack}}) = \prod_{j=1}^{n - n_i} \big((1 - b_j^{(i)})(1 - r_{\text{pack},j}) + b_j^{(i)} \cdot r_{\text{pack},j}\big).$$

This is the multilinear polynomial that evaluates to 1 on the subcube of $\tilde{f}^{(i)}$ and 0 elsewhere (it is the equality polynomial restricted to the prefix bits).

3. Both prover and verifier compute:

$$\text{packed\_eval} = \sum_i w_i(r_{\text{pack}}) \cdot v_i.$$

4. Register a single opening claim: $\tilde{f}(r_{\text{pack}}, r_x) = \text{packed\_eval}$.

This single claim is proved with one PCS opening proof.

### Soundness

If any individual claim $\tilde{f}^{(i)}(r_x) = v_i$ is false, then acceptance requires

$$0 = \sum_i w_i(r_{\text{pack}}) \cdot \Delta_i =: P(r_{\text{pack}}),$$

where $\Delta_i = \tilde{f}^{(i)}(r_x) - v_i$. Because the prefix code is prefix-free, the selector polynomials $\{w_i\}$ are linearly independent, so $P$ is a non-zero polynomial whenever some $\Delta_i \ne 0$. Since each variable in $r_{\text{pack}}$ appears with degree at most 1 in every $w_i$, Schwartz-Zippel gives:

$$\Pr[P(r_{\text{pack}}) = 0] \le \ell_{\text{pack}} / |\mathbb{F}_q|,$$

where $\ell_{\text{pack}} = \max_i (n - n_i)$ is the number of packing-prefix variables.

## Example: Packing Three Polynomials

Suppose we have:
- $\tilde{f}^{(1)}$ on $n_1 = 3$ variables ($2^3 = 8$ coefficients)
- $\tilde{f}^{(2)}$ on $n_2 = 2$ variables ($2^2 = 4$ coefficients)
- $\tilde{f}^{(3)}$ on $n_3 = 2$ variables ($2^2 = 4$ coefficients)

Total coefficients: $8 + 4 + 4 = 16$, so $n = \lceil \log_2 16 \rceil = 4$.

Prefix lengths: $l_1 = 4 - 3 = 1$, $l_2 = 4 - 2 = 2$, $l_3 = 4 - 2 = 2$.

Greedy assignment (sorted by $n_i$ descending):
- $\tilde{f}^{(1)}$: prefix $b^{(1)} = 0$ (1 bit), occupies subcube $\{0\} \times \{0,1\}^3$ (positions 0-7).
- $\tilde{f}^{(2)}$: prefix $b^{(2)} = 10$ (2 bits), occupies subcube $\{1,0\} \times \{0,1\}^2$ (positions 8-11).
- $\tilde{f}^{(3)}$: prefix $b^{(3)} = 11$ (2 bits), occupies subcube $\{1,1\} \times \{0,1\}^2$ (positions 12-15).

The packed hypercube:

```
Position:  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
           |--------- f^(1) ---------|  |-- f^(2) --|  |-- f^(3) --|
```

No padding, no overlap, no wasted space. One commitment, one opening proof.

## Concrete Sizing in the Jolt Recursion Paper

For the Dory verifier with $\sigma = 19$ reduce-and-fold rounds:

1. **Family packing** first extends each polynomial's domain by a constraint-index suffix of $k$ bits (where $k = \lceil \log_2 N \rceil$ and $N$ is the number of instances of that operation type). This groups all witness polynomials of the same type across instances into a single polynomial.

2. **Prefix packing** then packs all family-packed polynomials into a single $n_{\text{pack}} = 21$-variable dense polynomial ($\sim$2.1M coefficients).

The prover commits to this single packed polynomial via [Hyrax](../pcs/hyrax.md) over Grumpkin. The verifier checks a single opening claim. The Hyrax verification cost (two MSMs of size $\sim 2^{10.5}$) accounts for ~62% of the extended verifier's total cycle budget, and this cost grows only with $\sigma = O(\log T)$, not with the trace length $T$.

Prefix packing is a folklore technique, first formally written up in Kabir Peshawaria, "Batching evaluation claims without homomorphic commitments" (Manuscript, 2025). The Efficient Recursion paper (Section 3.3) repurposes it: where Peshawaria's goal was to minimise committed data for prover speed, the recursion paper uses the same technique to minimise *verifier* cost, since the Hyrax verifier time grows with $\sqrt{n_{\text{pack}}}$.

(*Efficient Recursion for the Jolt zkVM*, Section 3.3; Peshawaria, 2025)
