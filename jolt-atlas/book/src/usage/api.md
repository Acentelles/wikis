# API Reference

The core API is in `jolt-atlas-core`. The typical flow is:

1. Load and preprocess the model once.
2. Call `ONNXProof::prove` with one or more inputs.
3. Call `proof.verify` to check the result.

---

## Default Type Instantiation

Jolt Atlas is generic over the field, commitment scheme, and transcript. The default instantiation uses BN254:

```rust
use jolt_atlas_core::onnx_proof::{
    ONNXProof,
    Fr,               // BN254 scalar field (ark_bn254::Fr)
    HyperKZG,         // HyperKZG commitment scheme
    Blake2bTranscript, // Blake2b Fiat-Shamir transcript
    Bn254,            // BN254 elliptic curve
};
```

---

## Model Loading

```rust
use atlas_onnx_tracer::model::{Model, RunArgs};

let model = Model::load("path/to/model.onnx", &RunArgs::default());
```

`Model::load` parses the ONNX protobuf, builds the `ComputationGraph`, pads all tensor dimensions to the next power-of-two, and quantizes weights to fixed-point integers.

### RunArgs

`RunArgs` configures model loading with a builder pattern:

```rust
use atlas_onnx_tracer::model::RunArgs;

let run_args = RunArgs::new([("batch_size", 1), ("sequence_length", 128)])
    .set_scale(12)
    .with_padding(true)
    .with_pre_rebase_nonlinear(true);

let model = Model::load("path/to/model.onnx", &run_args);
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `variables` | `HashMap<String, usize>` | `{"batch_size": 1}` | Concretizes symbolic/dynamic ONNX dimensions (e.g. `batch_size`, `sequence_length`). Most transformer models have dynamic axes that must be resolved to fixed shapes before the graph can be compiled. |
| `scale` | `i32` | `7` | Fixed-point quantization denominator. Weights and activations are converted to integers as $\text{round}(x \cdot 2^{\text{scale}})$. Higher values give more precision but produce larger integers, increasing overflow risk and range-check circuit size. |
| `pad_to_power_of_2` | `bool` | `true` | Pads all tensor dimensions to the next power of two (e.g. `[3, 7, 15, 8]` becomes `[4, 8, 16, 8]`). Required because sumcheck and multilinear polynomial protocols operate over boolean hypercubes of size $2^n$. Disable only for inspection or debugging. |
| `pre_rebase_nonlinear` | `bool` | `false` | Workaround for `i32` overflow in Square/Cube ops on large models (GPT-2, BGE). Normally `Square(x) = x² / S` (compute then rebase). When enabled, decomposes to `x' = x/S; result = x' * x`, dividing *before* squaring to keep intermediates in `i32` range. Temporary until fused `i64`-precision ops become the default path. |

---

## Preprocessing

Preprocessing converts a loaded `Model` into the data structures needed by the prover and verifier. Three structs form a hierarchy, each adding commitment-scheme material on top of the shared model graph. This is a one-time cost per model.

```rust
use jolt_atlas_core::onnx_proof::{
    AtlasSharedPreprocessing, AtlasProverPreprocessing, AtlasVerifierPreprocessing,
    Fr, HyperKZG, Bn254,
};

// 1. Shared preprocessing: wraps the model graph
let shared_pp = AtlasSharedPreprocessing::preprocess(model);

// 2. Prover preprocessing: generates the SRS
let prover_pp = AtlasProverPreprocessing::<Fr, HyperKZG<Bn254>>::new(shared_pp);

// 3. Verifier preprocessing: derives the verification key from the SRS
let verifier_pp = AtlasVerifierPreprocessing::<Fr, HyperKZG<Bn254>>::from(&prover_pp);
```

| Step | Struct | What it does |
|------|--------|--------------|
| 1 | `AtlasSharedPreprocessing` | Wraps the `Model` and exposes `get_models_committed_polynomials()`, which walks every node in the computation graph to collect the committed polynomials each operator requires. Shared by both prover and verifier. |
| 2 | `AtlasProverPreprocessing` | Calls `model.max_num_vars()` to find the largest polynomial degree in the circuit, then runs `PCS::setup_prover(max_num_vars)` to generate the **SRS** (Structured Reference String), specifically the KZG trusted setup for HyperKZG with enough G1/G2 curve points. This is the expensive step. |
| 3 | `AtlasVerifierPreprocessing` | Derives the **verification key** from the prover setup via `PCS::setup_verifier()`. For HyperKZG this extracts a small subset of the SRS points needed to check opening proofs. Much smaller than the prover key, it can be serialized and distributed to untrusted verifiers independently of the model weights. |

---

## Proving

```rust
use atlas_onnx_tracer::tensor::Tensor;

// Prepare input tensor (i32 fixed-point)
let input = Tensor::new(Some(&data), &dims).unwrap();

// Generate proof
let (proof, io, debug_info) = ONNXProof::<Fr, Blake2bTranscript, HyperKZG<Bn254>>::prove(
    &prover_pp,
    &[input],
);
```

**Returns:**
- `proof: ONNXProof<F, T, PCS>`: the SNARK proof
- `io: ModelExecutionIO`: the public input/output pair (sent to the verifier)
- `debug_info: Option<ProverDebugInfo<F, T>>`: internal state for debugging (None in release builds)

---

## Verification

```rust
proof.verify(&verifier_pp, &io, None)?;
// Ok(()) on success, Err(ProofVerifyError) on failure
```

The verifier needs `verifier_pp` and `io`. While `verifier_pp` contains the model graph (via `AtlasSharedPreprocessing`), its commitment-scheme key is much smaller than the prover's: the `KZGProverKey` holds the full SRS (all G1/G2 powers, scaling with `max_num_vars`) whereas the `KZGVerifierKey` is just 4 constant-size curve points (`g1`, `g2`, `alpha_g1`, `beta_g2`). Model weights are not needed; the verifier checks them indirectly through the polynomial commitments in the proof.

---

## Tensor Construction

```rust
use atlas_onnx_tracer::tensor::Tensor;

// From a flat slice with explicit dims
let mut tensor = Tensor::new(Some(&[1, 2, 3, 4]), &[2, 2]).unwrap();

// Set the fixed-point scale (default: None)
// A value v with scale s represents the real number v / 2^s
tensor.set_scale(8); // 8-bit fixed point
```

---

## Generic Type Parameters

| Parameter | Trait bound | Default |
|-----------|-------------|---------|
| `F` | `JoltField` | `Fr` (BN254 scalar field, ~254-bit) |
| `T` | `Transcript` | `Blake2bTranscript` |
| `PCS` | `CommitmentScheme<Field = F>` | `HyperKZG<Bn254>` |

To swap the commitment scheme or transcript, change the type parameters:

```rust
use joltworks::transcripts::KeccakTranscript;

let (proof, io, _) = ONNXProof::<Fr, KeccakTranscript, HyperKZG<Bn254>>::prove(&prover_pp, &[input]);
```

`KeccakTranscript` produces EVM-compatible challenges, which is a prerequisite for on-chain verification.

---

## ONNXProof Structure

```rust
pub struct ONNXProof<F: JoltField, T: Transcript, PCS: CommitmentScheme<Field = F>> {
    /// Opening claims for committed polynomials
    pub opening_claims: Claims<F>,
    /// Sumcheck proofs keyed by (node_idx, ProofType)
    pub proofs: BTreeMap<ProofId, SumcheckInstanceProof<F, T>>,
    /// HyperKZG commitments to witness polynomials
    pub commitments: Vec<PCS::Commitment>,
    /// N-to-1 evaluation reduction proofs (one per node)
    pub eval_reduction_proofs: BTreeMap<usize, EvalReductionProof<F>>,
    // Reduced opening proof (private field)
}
```

See [Phase 3: Sumcheck DAG](../how/pipeline/sumcheck-dag.md) for a detailed explanation of each field.

---

## Serialization

`ONNXProof` implements `CanonicalSerialize` and `CanonicalDeserialize`. Use the `proof_serialization` module for binary encoding:

```rust
use jolt_atlas_core::onnx_proof::proof_serialization;

let bytes = proof_serialization::serialize_proof(&proof)?;
let proof: ONNXProof<Fr, Blake2bTranscript, HyperKZG<Bn254>> =
    proof_serialization::deserialize_proof(&bytes)?;
```
