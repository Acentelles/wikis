# Division and Rsqrt

Division is the canonical example of a "composite" operator: it requires pre-committed auxiliary polynomials (quotient and remainder) plus auxiliary sumchecks (range check and one-hot check) beyond the main execution sumcheck.

---

## What the Div Sumcheck Proves

For a `Div` node with left operand $a$, right operand $b$, and output $q$ (quotient), the constraint is:

$$a(x) = b(x) \cdot q(x) + R(x) \quad \forall x \in \{0,1\}^k$$

In sumcheck form:

$$\sum_{x \in \{0,1\}^k} \widetilde{eq}(r, x) \cdot \bigl(b(x) \cdot q(x) + R(x) - a(x)\bigr) = 0$$

This is a degree-3 polynomial (degree 1 from $\widetilde{eq}$, degree 2 from $b \cdot q$).

The remainder $R$ is a committed polynomial (`DivNodeQuotient` actually refers to the output $q$; the remainder is `DivRemainder`, a virtual polynomial). Without an additional range check, a cheating prover could use an $R$ outside $[0, b)$.

---

## Committed Polynomials

```rust
fn get_committed_polynomials(node) -> Vec<CommittedPolynomial> {
    let mut polys = vec![CommittedPolynomial::DivNodeQuotient(node.idx)];
    // Plus one chunk per dimension of the range-check address encoding
    polys.extend((0..d).map(|i| CommittedPolynomial::DivRangeCheckRaD(node.idx, i)));
    polys
}
```

- `DivNodeQuotient`: the output tensor $q$ as a dense MLE.
- `DivRangeCheckRaD(node_idx, d)`: one-hot lookup address for the $d$-th chunk of the range-check encoding (see [Range Checking](../lookups/range-checking.md)).

---

## DivParams: Drawing the Evaluation Point

Before any sumcheck message is sent, `DivParams::new` draws the evaluation point from the transcript:

```rust
let params = DivParams::new(node.clone(), &mut prover.transcript);
// Draws r ∈ F^k from transcript
```

This is the point $r$ at which the verifier will check the final sumcheck claim. Fixing $r$ before any sumcheck round message is sent ensures the prover cannot adapt the polynomial to the challenge.

---

## DivProver: Initializing the MLEs

```rust
let mut exec_sumcheck = DivProver::initialize(&prover.trace, params);
```

This materializes five MLEs from the trace:
- `left_operand`: MLE of $a$
- `right_operand`: MLE of $b$
- `q`: MLE of the output (quotient)
- `R`: MLE of the remainder, computed as `a % b` (with sign correction for signed integers)
- `eq_r_node_output`: the Gruen split equality polynomial $\widetilde{eq}(r, \cdot)$

A debug assertion checks that $\sum_i (b_i \cdot q_i + R_i - a_i) = 0$ over all $2^k$ entries.

---

## The Sumcheck Loop

`Sumcheck::prove` runs $k$ rounds. Each round:

1. `compute_message(j)`: computes a degree-3 univariate $h_j(X)$ using Gruen's optimization:
   ```
   c0 = b(2g) * q(2g) + R(2g) - a(2g)
   e  = (b(2g+1) - b(2g)) * (q(2g+1) - q(2g))  // quadratic cross-term
   ```
   These fold into `[q_constant, q_quadratic]` then combined with the equality polynomial via `gruen_poly_deg_3`.

2. Appended to transcript → challenge $r_j$ drawn.

3. `ingest_challenge(r_j)`: binds all five MLEs on variable $j$ in parallel:
   ```rust
   self.eq_r_node_output.bind(r_j);
   self.left_operand.bind_parallel(r_j, LowToHigh);
   self.right_operand.bind_parallel(r_j, LowToHigh);
   self.q.bind_parallel(r_j, LowToHigh);
   self.R.bind_parallel(r_j, LowToHigh);
   ```

After $k$ rounds each MLE has been reduced to a scalar: `left_operand.final_sumcheck_claim()` = $a(r_\text{sumcheck})$, and so on.

---

## After the Loop: cache_openings

```rust
accumulator.append_virtual(transcript,
    VirtualPolynomial::NodeOutput(inputs[0]),  // left operand
    SumcheckId::NodeExecution(node.idx),
    opening_point,
    self.left_operand.final_sumcheck_claim(),
);
// similarly for right_operand, q (= NodeOutput(node.idx)), R (= DivRemainder)
```

"Virtual" means the polynomial has no direct commitment; it will be opened indirectly through the eval-reduction chain. The exception is `q`, which is linked to the pre-committed `DivNodeQuotient`.

---

## Verifier Check

The verifier reconstructs the expected final sumcheck claim:

```rust
expected = eq_eval * (right_operand_claim * q_claim + R_claim - left_operand_claim)
```

If `final_claim != expected`, the sumcheck verification fails.

---

## ScalarConstDiv

`ScalarConstDiv` divides every element by a known scalar constant. The constraint simplifies because the divisor is public. The committed polynomial is `ScalarConstDivNodeRemainder` (the remainder MLE). The range check uses the constant divisor value directly.

---

## Rsqrt

`Rsqrt` (reciprocal square root) uses a similar pattern:

1. The prover commits `RsqrtNodeInv`, the inverse $1/\sqrt{x}$ for each element.
2. The execution sumcheck proves $x \cdot \text{inv}^2 = 1$ (degree 4).
3. A range check proves $\text{inv}$ is in the valid range for reciprocal square roots.
4. A one-hot check proves the range-check address is valid.
