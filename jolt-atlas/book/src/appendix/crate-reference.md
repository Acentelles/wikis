# Crate Reference

Quick reference for the four workspace crates in the `jolt-atlas` repository.

---

## common

**Purpose:** Shared constants and type definitions used across all crates.

**Key exports:**

```rust
// Polynomial naming enums (used as keys in BTreeMaps)
pub enum CommittedPolynomial {
    NodeOutputRaD(usize, usize),
    DivNodeQuotient(usize),
    DivRangeCheckRaD(usize, usize),
    TeleportNodeQuotient(usize),
    TeleportRangeCheckRaD(usize, usize),
    SoftmaxRemainder(usize, usize),
    SoftmaxExponentiationRaD(usize, usize),
    GatherRa(usize),
    // ... (all committed polynomial variants)
}

pub enum VirtualPolynomial {
    NodeOutput(usize),
    DivRemainder(usize),
    TeleportQuotient(usize),
    SoftmaxExponentiation(usize, usize),
    // ...
}

// Constants
pub const LOG_K_CHUNK: usize = 4;
pub const K_CHUNK: usize = 16;
pub const LOG_K: usize = 64;
pub const XLEN: usize = 32;
```

**Dependencies:** None (no workspace crate dependencies).

---

## joltworks

**Purpose:** Cryptographic primitives: fields, polynomials, sumcheck, commitment schemes, transcripts, lookup tables.

**Key exports:**

```rust
// Field
pub trait JoltField: PrimeField + JoltFieldExt;

// Polynomials
pub enum MultilinearPolynomial<F> { Dense(DensePolynomial<F>), OneHot(OneHotPolynomial) }
pub struct DensePolynomial<F> { Z: Vec<F>, num_vars: usize }
pub struct OneHotPolynomial { indices: Vec<u16>, num_vars: usize }
pub struct GruenSplitEqPolynomial<F> { ... }
pub struct UniPoly<F> { coeffs: Vec<F> }
pub struct CompressedUniPoly<F> { coeffs_except_linear_term: Vec<F> }

// Commitment schemes
pub trait CommitmentScheme { type Field; type Commitment; type Proof; ... }
pub struct HyperKZG<C: Pairing>;
pub struct HyperKZGSRS<C: Pairing>;
pub struct HyperKZGProverKey<C>;
pub struct HyperKZGVerifierKey<C>;

// Sumcheck
pub struct SumcheckInstanceProof<F, T>;
pub struct BatchedSumcheck;
pub trait SumcheckInstanceProver<F, T>;
pub trait SumcheckInstanceVerifier<F, T>;
pub struct ProverOpeningAccumulator<F>;
pub struct VerifierOpeningAccumulator<F>;
pub struct EvalReductionProof<F>;

// Transcripts
pub trait Transcript;
pub struct Blake2bTranscript;
pub struct KeccakTranscript;

// Errors
pub struct ProofVerifyError;
```

**Dependencies:** `common`, Arkworks (BN254), Rayon, Serde, Blake2, SHA3.

---

## atlas-onnx-tracer

**Purpose:** ONNX model loading, quantization, forward-pass execution. Independent of ZK.

**Key exports:**

```rust
// Model
pub struct Model { graph: ComputationGraph, ... }
impl Model {
    pub fn load(path: &str, opts: &ModelLoadOptions) -> Result<Self>;
    pub fn trace(&self, inputs: &[Tensor<i32>]) -> Trace;
    pub fn nodes(&self) -> &BTreeMap<usize, ComputationNode>;
}

// Graph
pub struct ComputationGraph {
    pub nodes: BTreeMap<usize, ComputationNode>,
    pub inputs: Vec<usize>,
    pub outputs: Vec<usize>,
    pub original_input_dims: HashMap<usize, Vec<usize>>,
    pub original_output_dims: HashMap<usize, Vec<usize>>,
}

pub struct ComputationNode {
    pub idx: usize,
    pub operator: Operator,
    pub inputs: Vec<usize>,
    pub output_dims: Vec<usize>,
}

pub enum Operator {
    Add, Sub, Mul, Div, Neg, Square, Cube,
    ReLU, Tanh, Sigmoid, Erf, Sin, Cos,
    Einsum(String), Softmax(usize),
    Gather, Reshape, Concat, Slice, MoveAxis,
    ReduceSum, ReduceMean, ReduceMax,
    Input(Tensor<i32>), Constant(Tensor<i32>),
    // ...
}

// Trace
pub struct Trace { pub node_outputs: BTreeMap<usize, Tensor<i32>> }
pub struct ModelExecutionIO { pub inputs: Vec<Tensor<i32>>, pub outputs: Vec<Tensor<i32>> }

// Tensor
pub struct Tensor<T> { data: Vec<T>, dims: Vec<usize>, scale: Scale }
impl<T> Tensor<T> {
    pub fn new(data: Vec<T>, dims: Vec<usize>) -> Self;
    pub fn set_scale(self, scale: Scale) -> Self;
    pub fn scale(&self) -> Scale;
    pub fn par_enum_map<U>(&self, f: impl Fn(usize, T) -> U) -> Tensor<U>;
}

// Quantization
pub type Scale = i32;
pub fn quantize_tensor(tensor: Tensor<f32>, scale: Scale) -> Tensor<i32>;
```

**Dependencies:** `common`, Tract (ONNX parsing), Rayon, Serde.

---

## jolt-atlas-core

**Purpose:** ZK prover and verifier for ONNX models. Depends on all other crates.

**Key exports:**

```rust
// Main proof API
pub struct ONNXProof<F, T, PCS> {
    pub opening_claims: Claims<F>,
    pub proofs: BTreeMap<ProofId, SumcheckInstanceProof<F, T>>,
    pub commitments: Vec<PCS::Commitment>,
    pub eval_reduction_proofs: BTreeMap<usize, EvalReductionProof<F>>,
}
impl<F, T, PCS> ONNXProof<F, T, PCS> {
    pub fn prove(pp: &AtlasProverPreprocessing<F, PCS>, inputs: &[Tensor<i32>])
        -> (Self, ModelExecutionIO, Option<ProverDebugInfo<F, T>>);
    pub fn verify(&self, pp: &AtlasVerifierPreprocessing<F, PCS>, io: &ModelExecutionIO, ...)
        -> Result<(), ProofVerifyError>;
}

// Preprocessing
pub struct AtlasSharedPreprocessing<F, PCS>;
impl AtlasSharedPreprocessing<F, PCS> {
    pub fn preprocess<PCS: CommitmentScheme>(model: &Model) -> Self;
}
pub struct AtlasProverPreprocessing<F, PCS>;
pub struct AtlasVerifierPreprocessing<F, PCS>;

// Proof identifiers
pub type ProofId = (usize, ProofType);
pub enum ProofType {
    Execution, RangeCheck, RaOneHotChecks,
    NeuralTeleport, SoftmaxDivSumMax,
    SoftmaxExponentiationReadRaf, SoftmaxExponentiationRaOneHot,
}

// Default type aliases (re-exported for convenience)
pub use ark_bn254::{Bn254, Fr};
pub use joltworks::{poly::commitment::hyperkzg::HyperKZG, transcripts::Blake2bTranscript};
```

**Dependencies:** `common`, `joltworks`, `atlas-onnx-tracer`, Arkworks (BN254), Rayon, Serde.
