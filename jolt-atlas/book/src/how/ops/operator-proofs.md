# Operator Proof Strategies

Every operator in the computation graph has a corresponding proof implementation in `jolt-atlas-core/src/onnx_proof/ops/`. This page explains the common interface and the taxonomy of proof strategies.

---

## The OperatorProofTrait

Every operator implements the `OperatorProofTrait`:

```rust
pub trait OperatorProofTrait<F: JoltField, T: Transcript> {
    type Params: SumcheckInstanceParams<F>;
    type Prover: SumcheckInstanceProver<F, T>;
    type Verifier: SumcheckInstanceVerifier<F, T>;

    fn prove(node: &ComputationNode, prover: &mut Prover<F, T>)
        -> Vec<(ProofId, SumcheckInstanceProof<F, T>)>;

    fn verify(node: &ComputationNode, verifier: &mut Verifier<F, T>)
        -> Result<(), ProofVerifyError>;

    fn get_committed_polynomials(node: &ComputationNode)
        -> Vec<CommittedPolynomial>;
}
```

The three associated types (`Params`, `Prover`, `Verifier`) implement the per-round sumcheck logic for that operator.

---

## The Standard Sumcheck Macro

Simple operators (Add, Sub, Mul, and most shape operators) use a macro that generates the full boilerplate:

```rust
impl_standard_params!(AddParams, degree: 2);
impl_standard_sumcheck_proof_api!(Add, AddParams, AddProver, AddVerifier);
```

`impl_standard_params!` declares the degree and number of rounds. `impl_standard_sumcheck_proof_api!` generates `prove`, `verify`, and `get_committed_polynomials` (returning `vec![]`) from the Params/Prover/Verifier triple.

---

## ReductionFlow

Each operator declares how its opening claims should be reduced after the sumcheck:

```rust
pub enum ReductionFlow {
    /// Default: use the standard eval-reduction for the node's output.
    Default,
    /// Custom: the operator handles its own eval-reduction and claim registration.
    Custom,
}
```

Simple operators use `ReductionFlow::Default`. Operators with pre-committed auxiliary polynomials (Div, Rsqrt, Neural Teleport, Softmax, Gather) use `ReductionFlow::Custom` to also link the pre-committed polynomial to the batch opening.

---

## Operator Taxonomy

### Category 1: Pure Sumcheck (no auxiliary polynomials)

These operators generate a single `ProofType::Execution` sumcheck. They return `vec![]` from `get_committed_polynomials`.

| Operator | Degree | Constraint |
|----------|--------|-----------|
| Add | 2 | $\text{out}(x) = \text{left}(x) + \text{right}(x)$ |
| Sub | 2 | $\text{out}(x) = \text{left}(x) - \text{right}(x)$ |
| Mul | 3 | $\text{out}(x) = \text{left}(x) \cdot \text{right}(x)$ |
| Square | 3 | $\text{out}(x) = \text{in}(x)^2$ |
| Cube | 4 | $\text{out}(x) = \text{in}(x)^3$ |
| Einsum | 3 | sum over contraction index $k$ |
| Reshape / MoveAxis / Slice / Concat | 2 | identity or selection constraint |
| Sum | 2 | reduction constraint |

### Category 2: Composite with Auxiliary Polynomials

These operators commit quotients, remainders, or inverse polynomials, and generate multiple `ProofType` entries.

- **[Div / ScalarConstDiv / Rsqrt](./div-rsqrt.md)**: Euclidean division with range check.

### Category 3: Lookup-Based Activations

These operators prove non-linear functions via lookup arguments.

- **[ReLU](./relu.md)**: One-hot Shout lookup.
- **[Neural Teleport (Tanh, Erf, Sigmoid, Sin, Cos)](./neural-teleport.md)**: Bounded-domain lookup via periodicity.

### Category 4: Composite Multi-Stage

- **[Softmax](./softmax.md)**: Division + exponentiation lookup + sum/max.
- **[Gather](./gather.md)**: Embedding table lookup via dense address polynomial.

---

## Proof Entries Per Node

| Operator | ProofType entries |
|----------|------------------|
| Add / Sub / Mul / Einsum / Reshape / … | `Execution` |
| ReLU | `Execution`, `RaOneHotChecks` |
| Div / Rsqrt | `Execution`, `RangeCheck`, `RaOneHotChecks` |
| Tanh / Erf / Sigmoid / Sin / Cos | `NeuralTeleport`, `RangeCheck`, `RaOneHotChecks` |
| Softmax | `SoftmaxDivSumMax`, `SoftmaxExponentiationReadRaf`, `SoftmaxExponentiationRaOneHot` |
| Gather | `Execution`, `RaOneHotChecks` |
