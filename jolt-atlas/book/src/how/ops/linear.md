# Linear Operators

Linear operators are the simplest category: a single sumcheck instance, no auxiliary committed polynomials, and no lookup arguments. The proof consists of verifying that the output multilinear extension (MLE) equals the claimed combination of input MLEs.

---

## Add and Sub

The constraint for `Add` is:

$$\sum_{x \in \{0,1\}^k} \widetilde{eq}(r, x) \cdot \bigl(\text{left}(x) + \text{right}(x) - \text{out}(x)\bigr) = 0$$

where $r$ is drawn from the transcript before the sumcheck begins. This is a degree-2 polynomial in each variable (degree 1 from $\widetilde{eq}$, degree 1 from the linear combination). The sumcheck produces 2-coefficient round messages.

`Sub` is identical with a sign flip on the right operand.

---

## Mul

```rust
impl_standard_params!(MulParams, degree: 3);
```

The constraint is:

$$\sum_{x \in \{0,1\}^k} \widetilde{eq}(r, x) \cdot \bigl(\text{left}(x) \cdot \text{right}(x) - \text{out}(x)\bigr) = 0$$

Degree 3 (1 from $\widetilde{eq}$, 2 from the product). Round messages have 4 coefficients.

---

## Square and Cube

`Square` proves $\text{out}(x) = \text{in}(x)^2$ (degree 3).  
`Cube` proves $\text{out}(x) = \text{in}(x)^3$ (degree 4).

Both use the standard macro. No auxiliary polynomials are needed because the input and output are both outputs of adjacent nodes, already linked via the eval-reduction chain.

---

## Einsum (See Dedicated Page)

Einsum is technically in this category (single sumcheck, no lookup), but it is more complex. See [Einsum and MatMul](./einsum.md).

---

## Shape Operators

Shape operators (Reshape, MoveAxis, Transpose, Slice, Concat, Broadcast, Identity) rearrange elements without computing new values. Their proof verifies that the output MLE is a permutation or selection of the input MLE.

**Example (Reshape):** The output tensor has a different shape but the same underlying flat data. The constraint is simply:

$$\sum_{x \in \{0,1\}^k} \widetilde{eq}(r, x) \cdot \bigl(\text{out}(x) - \text{in}(x)\bigr) = 0$$

The equality is trivial since reshaping preserves flat ordering. The sumcheck still needs to run to link the output MLE claim to the input MLE opening point.

**Concat:** The constraint selects the appropriate half of the input domain for each output position, using an indicator function derived from the concatenation axis.

**Broadcast:** Maps each output index to the corresponding input index, proving the output is a valid repetition of the input.

---

## Reduction Operators: Sum

`Sum` reduces a tensor along one or more axes. The constraint is:

$$\text{out}[i] = \sum_{j} \text{in}[i, j]$$

expressed as a sumcheck over the reduction index $j$ with the equality polynomial restricting to the output index $i$.

---

## Iff (Conditional Select)

`Iff` implements $\text{out}(x) = \text{cond}(x) \cdot \text{true\_val}(x) + (1 - \text{cond}(x)) \cdot \text{false\_val}(x)$. This is degree 3 (the product `cond * true_val`).

---

## Clamp

`Clamp` clips values to $[\text{min}, \text{max}]$. It is proved as a lookup: each element is looked up in a clamp table that maps any `i32` input to its clamped `i32` output. The lookup argument follows the same one-hot pattern as ReLU.

---

## Neg

`Neg` proves $\text{out}(x) = -\text{in}(x)$. Degree 2. No auxiliary polynomials.

---

## Empty get_committed_polynomials

All operators in this category return `vec![]`:

```rust
fn get_committed_polynomials(&self, _node: &ComputationNode) -> Vec<CommittedPolynomial> {
    vec![]
}
```

This means no pre-commitment step is required and the IOP loop visits them with zero transcript messages before the first sumcheck challenge.
