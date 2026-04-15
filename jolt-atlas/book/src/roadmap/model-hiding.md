# Hiding the Model

This page answers a precise question: **what does the verifier learn at each stage of the Jolt Atlas protocol, and what would need to change to hide the model?**

We address two levels of hiding: **weight hiding** (the model architecture is public but the learned parameters are private) and **architecture hiding** (the full model structure is also private). Weight hiding is the nearer-term goal; architecture hiding is more challenging but recent research has demonstrated feasible approaches.

The treatment is grounded in the codebase as it stands. Every claim about what is or is not public is traceable to a specific struct, field, or function call.

---

## What the Verifier Sees Today

Jolt Atlas currently operates in a fully public mode. There is no separation between what the prover knows and what the verifier knows about the model.

### 1. Preprocessing

The verifier is initialized with `AtlasVerifierPreprocessing<F, PCS>` (defined in `jolt-atlas-core/src/onnx_proof/preprocessing.rs`):

```rust
pub struct AtlasVerifierPreprocessing<F, PCS> {
    pub generators: PCS::VerifierSetup,   // HyperKZG verification key
    pub shared: AtlasSharedPreprocessing, // ← the model, fully public
}

pub struct AtlasSharedPreprocessing {
    pub model: Model,                     // ← full ComputationGraph
}
```

`AtlasSharedPreprocessing` is shared by both prover and verifier. It holds the complete `Model`, which contains:

```rust
pub struct Model {
    pub graph: ComputationGraph,
}
pub struct ComputationGraph {
    pub nodes: BTreeMap<usize, ComputationNode>,
    pub inputs: Vec<usize>,
    pub outputs: Vec<usize>,
    ...
}
pub struct ComputationNode {
    pub idx: usize,
    pub operator: Operator,           // ← operator type + all operator-specific fields
    pub inputs: Vec<usize>,           // ← DAG edges
    pub output_dims: Vec<usize>,      // ← tensor shapes (padded to power-of-two)
}
```

The `Operator` enum includes a `Constant` variant that embeds the full weight tensor directly:

```rust
pub enum Operator {
    Constant(Tensor<i32>),  // ← every weight matrix, bias, embedding table
    Einsum { equation: String, scale: i32 },
    Tanh { scale: F32, tau: i32, log_table: usize },
    // ...
}
```

**Everything in `AtlasSharedPreprocessing` is public to the verifier before the proof even begins.**

### 2. Model I/O

The `ONNXProof::verify` call signature is:

```rust
pub fn verify(
    &self,
    pp: &AtlasVerifierPreprocessing<F, PCS>,
    io: &ModelExecutionIO,   // ← inputs and outputs in the clear
    ...
) -> Result<(), ProofVerifyError>
```

`ModelExecutionIO` contains the quantized input tensor(s) and output tensor(s). These are public by default in the current design.

### 3. The Proof Object

`ONNXProof<F, T, PCS>` contains:

```rust
pub struct ONNXProof<F, T, PCS> {
    pub opening_claims: Claims<F>,                          // evaluation claims
    pub proofs: BTreeMap<ProofId, SumcheckInstanceProof>,   // sumcheck round messages
    pub commitments: Vec<PCS::Commitment>,                  // HyperKZG commitments
    pub eval_reduction_proofs: BTreeMap<usize, EvalReductionProof<F>>,
    reduced_opening_proof: Option<ReducedOpeningProof<...>>,
}
```

The commitments are opaque curve points. The sumcheck round messages are field elements that correspond to polynomial evaluations on a random path; they reveal nothing about the polynomial's coefficients. The verifier never receives any intermediate activation tensor in plaintext.

### Summary table

| Information | Visible to verifier? | Where |
|---|---|---|
| Operator types and their parameters (scale, tau, equation, …) | **Yes** | `AtlasSharedPreprocessing` → `ComputationNode.operator` |
| Graph connectivity and tensor shapes | **Yes** | `AtlasSharedPreprocessing` → `ComputationNode.inputs`, `output_dims` |
| Weight tensors (`Constant` nodes) | **Yes** | `AtlasSharedPreprocessing` → `Operator::Constant(Tensor<i32>)` |
| Model inputs | **Yes** | `ModelExecutionIO.inputs` |
| Model outputs | **Yes** | `ModelExecutionIO.outputs` |
| Intermediate activation tensors | **No** | Held only in `Trace`, committed or used inside sumcheck |
| Witness polynomial coefficients (quotients, RaD addresses) | **No** | Committed; only `Vec<PCS::Commitment>` leaves the prover |
| Sumcheck polynomial coefficients | **No** | Only field evaluations at challenge points appear in the proof |

---

## Architecture vs. Weights

Before describing what can be hidden, it is important to distinguish two categories within the model:

### Architecture

Architecture is everything about *how the computation is structured*:

- The set and order of operator types: `Add`, `Einsum`, `ReLU`, `Tanh`, `SoftmaxAxes`, etc.
- The DAG connectivity: which node feeds which.
- Tensor shapes at every node (`output_dims`).
- Operator hyperparameters that govern the proof structure: `tau` and `log_table` for neural-teleport activations (Tanh, Erf, Sigmoid), `equation` for Einsum, `axes` for Softmax, `divisor` for ScalarConstDiv.

These fields determine how the verifier executes the IOP: which committed polynomials to expect, what sumcheck degree to use, how many variables appear in each sumcheck, and which lookup tables are active. The verifier cannot perform any check without them.

### Weights

Weights are the *data* tensors carried by `Operator::Constant` nodes, the values that do not change across inference calls. In a transformer these include:

- Embedding tables (token and positional)
- Query/Key/Value projection matrices (`W_Q`, `W_K`, `W_V`)
- Output projection matrices (`W_O`)
- Feed-forward weight matrices (`W_1`, `W_2`)
- Layer-norm scale and bias parameters
- Output projection (LM head)

All of these arrive as `Operator::Constant(Tensor<i32>)` nodes in the `ComputationGraph`. They are numerically dense and represent the model's intellectual property.

---

## Architecture Hiding: Challenges

The verifier invokes the model structure at multiple points in `ONNXProof::verify`:

1. **`AtlasVerifierPreprocessing::from`** calls `PCS::setup_verifier`, whose key size depends on `model.max_num_vars()`. That number is derived from operator types and tensor shapes.

2. **`verify_output_claim`** calls `output_computation_node.pow2_padded_num_output_elements().log_2()` to draw the right-length challenge vector `τ`.

3. **`verify_iop`** iterates `model.graph.nodes` in reverse and calls `OperatorVerifier::verify(node, verifier, eval_reduction_proofs)`. This dispatches through `dispatch_operator!`, which pattern-matches on `node.operator`. The verifier must know the operator variant to run the correct verification logic.

4. **`get_committed_polynomials`** (called to confirm the number of expected commitments) dispatches the same way.

5. **Operator-specific checks**: for example, the Einsum verifier reconstructs the expected MLE dimension from the `equation` field; the Tanh verifier uses `inner.tau` and `inner.log_table` to derive the correct lookup table size.

In the current design, hiding the architecture would require replacing all of these concrete lookups with verifiable commitments. The sections below describe how weight hiding and architecture hiding can each be achieved.

---

## Hiding the Weights: What Changes

The goal is to remove `Operator::Constant(Tensor<i32>)` data from `AtlasSharedPreprocessing` and replace it with per-weight HyperKZG commitments. The verifier gains confidence that the prover used a committed weight, but learns no weight values.

### New preprocessing split

```rust
/// Published once. Shared between prover and verifier.
/// Contains the full architecture but no weight data.
pub struct ModelKey<F, PCS> {
    pub architecture: AtlasSharedPreprocessing,  // Constant nodes stripped of data
    pub weight_commitments: Vec<PCS::Commitment>, // one commitment per Constant node
}

/// Held only by the prover. Never transmitted.
pub struct ModelSecretKey<F> {
    pub weight_polynomials: Vec<MultilinearPolynomial<F>>,
    pub blinding_randomness: Vec<F>,
}
```

`AtlasSharedPreprocessing.model` would replace each `Operator::Constant(tensor)` with `Operator::Constant(shape_only)`, keeping the shape (needed for architecture reasoning) but discarding the data. The actual weight MLEs live in `ModelSecretKey`, which never leaves the prover.

### Proof generation changes

`ONNXProof::prove` currently calls `commit_witness_polynomials` which commits only to *auxiliary* witness polynomials (quotients, RaD address polynomials, etc.). In the non-hiding case, simple operators like Add and Einsum need no commitments because their sumchecks only reference node outputs that are already constrained by the eval-reduction chain (see [CommittedPolynomial](../how/pipeline/sumcheck-dag.md#committedpolynomial)). With hidden weights this changes: weight polynomials are no longer public constants, so they become prover-chosen values that must be committed before any challenge is drawn, just like quotients and lookup addresses. In effect, *every* operator that touches a weight tensor (Einsum, Gather, Add with bias) now requires a commitment. A parallel step commits to each weight polynomial:

```
// Step 0 (new): commit to all weight polynomials
for each Constant node in model.graph.nodes:
    weight_poly = MultilinearPolynomial::from(Constant node tensor)
    com = HyperKZG::commit(weight_poly)
    transcript.append_serializable(com)           // binds before any challenge
    weight_commitments.push(com)
    weight_polynomials_map.insert(node_idx, weight_poly)

// Step 1 (unchanged): commit to witness polynomials (quotients, RaD, etc.)
commit_witness_polynomials(...)
```

The critical ordering invariant is preserved: all weight commitments precede all IOP challenges.

### IOP loop changes

During the IOP loop, operators that use weight data (principally `Einsum`, `Gather`, and linear operators with constant addends) currently access the weight tensor directly from the `Operator::Constant` node. With hiding, they instead register an *opening claim* against the weight commitment:

**Einsum (current):** The sumcheck proves `O(r) = Σ_x W(x) · I(r, x)`. The prover evaluates W at the sumcheck challenge point explicitly, and the verifier does the same from its public copy of W.

**Einsum (with hidden weights):** The prover still evaluates `W(r_x)` internally (it has the weight polynomial). It appends this evaluation to the `ProverOpeningAccumulator` as an opening claim against the committed weight polynomial, exactly as it already does for other committed polynomials (quotients, etc.). The verifier's sumcheck check becomes: *"given the claimed value `W(r_x) = v`, is the sumcheck consistent?"* The actual binding check (that `v` matches the committed W) is deferred to the batch opening stage.

**Gather (embedding lookup):** The embedding table is a `Constant` node. Its MLE polynomial would be committed as a weight. The `GatherRa` address polynomial is already committed as a witness polynomial (it already appears in `CommittedPolynomial::GatherRa`). The binding between address and table value would require an opening claim on the embedding table commitment at the relevant challenge point.

Operators with no weight data (`ReLU`, `Add`, `Sub`, `Tanh`, `Sigmoid`, `Erf`, `Sin`, `Cos`, `Softmax`, `Div`, `Rsqrt`, `ScalarConstDiv`) require no changes.

### Batch opening changes

`prove_reduced_openings` currently opens all *witness* polynomials at the joint batch challenge point. With hidden weights, the weight polynomials enter the same accumulator:

```
// Unified poly_map (weights + witnesses)
let all_polys = weight_polynomials_map
    .into_iter()
    .chain(witness_poly_map.into_iter())
    .collect::<BTreeMap<_, _>>();

// Unchanged batch opening machinery:
accumulator.prepare_for_sumcheck(&all_polys);
let (batch_sumcheck_proof, r_batch) = accumulator.prove_batch_opening_sumcheck(&mut transcript);
let rlc = build_materialized_rlc(&gamma_powers, &all_polys);
let joint_opening = HyperKZG::prove(generators, &rlc, &r_batch, ...);
```

The verifier-side `verify_reduced_openings` replaces `PCS::combine_commitments(&self.commitments, ...)` with a call that combines *all* commitments (both witness and weight) using the same gamma-power RLC. This requires no new cryptographic machinery; the structure of `ReducedOpeningProof` is unchanged.

### What the verifier learns about hidden weights

With HyperKZG, the verifier learns:
- A commitment `com_W` (a single $\mathbb{G}_1$ element per weight polynomial).
- One multilinear evaluation: `W(r) = v` at a random point `r ∈ F^k`.

The evaluation point `r` is derived from the Fiat-Shamir transcript after all commitments are fixed. Under the discrete-log assumption, a single evaluation at a uniformly random point is computationally indistinguishable from a random field element. The verifier cannot recover any individual weight coefficient or reconstruct any weight row from this.

---

## Per-Operator Analysis

The following table categorises each operator by what work is needed to support weight hiding.

| Operator | Has weight data? | Change needed |
|---|---|---|
| `Constant` | Yes (the weight itself) | Commit weight polynomial; remove data from shared preprocessing |
| `Einsum` | Via `Constant` inputs | Replace direct W evaluation with opening claim against weight commitment |
| `Gather` | Via `Constant` input (embedding table) | Commit embedding table; add opening claim for table at challenge point |
| `Add`, `Sub`, `Mul` with bias | Via `Constant` input | Commit bias polynomial; add opening claim |
| `ReLU` | No | No change |
| `Tanh`, `Erf`, `Sigmoid`, `Sin`, `Cos` | No (tau, log_table are architectural constants) | No change |
| `SoftmaxAxes` | No | No change |
| `Div`, `Rsqrt`, `ScalarConstDiv` | No (divisor is architectural) | No change |
| `Reshape`, `Concat`, `Slice`, `MoveAxis`, `Identity` | No | No change |
| `Sum`, `Square`, `Cube`, `Neg`, `Clamp` | No | No change |

---

## What Remains Public with Weight Hiding Only

With weight hiding alone, the verifier still learns:

1. **The architecture** in full: operator types, connectivity, shapes, quantization hyperparameters.
2. **The number of weight tensors and their shapes**: the shape is needed to size the HyperKZG commitment and to know `max_num_vars`.
3. **One evaluation of each weight MLE** at a uniformly random challenge point. This is a single field element per weight polynomial and reveals nothing recoverable under standard assumptions.
4. **The model inputs and outputs** (unless input/output hiding is also implemented; see [Security Considerations](../appendix/security.md)).

For many use cases this is sufficient: knowing that a model is a transformer with specific dimensions does not reveal the policy it encodes. But for full model confidentiality, the architecture must also be hidden.

---

## Hiding the Architecture via Recursive Verification

Architecture hiding is substantially harder than weight hiding because the verifier's IOP logic is currently driven by the model graph. The five verifier touch-points listed in [Architecture Hiding: Challenges](#architecture-hiding-challenges) all pattern-match on operator types and tensor shapes. Rather than redesigning the verifier to work without seeing the model structure, we can use the approach described in the zkARc paper: **prove that the verifier ran correctly, with the model as a private witness**.

### The recursive verification approach

In a standard zkSNARK, the verifier $V$ takes:
- The program $M$ (e.g., the ONNX model),
- A proof $\pi_M$ that the program was executed correctly,
- Commitments to the witnesses $\bar{\omega}$,
- The public inputs and outputs $x$.

The model $M$ is public because $V$ needs it to construct the constraint system. To hide $M$, we add a second proving layer:

```
Step 1:  P_M proves the model M, producing π_M
              (standard Jolt Atlas proof, model is known to P_M)

Step 2:  P_V proves that V(M, π_M, ω̄, x) = accept
              with M as a private witness
              producing π_V and a commitment M̄ = Com(M)

Step 3:  V' verifies π_V using only M̄ and x
              V' never sees M, π_M, or the sumcheck transcripts
```

The recursive verifier $V'$ checks that *some* model was verified successfully, learning only the commitment $\bar{M}$ and the public I/O. The model itself, its architecture, and all intermediate proof artifacts are hidden inside the witness of $\pi_V$.

### Why this works naturally with a DAG of sumchecks

Jolt Atlas already structures its proof as a DAG of sumcheck instances, and the Vega/NovaBlindFold technique defers all sumcheck verification claims to a single small R1CS instance. The Jolt Atlas verifier performs:

1. Derive challenges from round commitments via the Fiat-Shamir transcript.
2. Construct the verifier R1CS deterministically from the model's stage configurations and public inputs.
3. Reconstruct the real instance from round commitments (setting $u = 1$, $E = 0$).
4. Absorb the random instance included in the proof.
5. Fold the real and random instances using cross-term commitments and challenge $r$.
6. Verify Spartan's outer sumcheck ($\log m$ rounds, degree-3 polynomials).
7. Verify Spartan's inner sumcheck ($\log |W|$ rounds, degree-2 polynomials).
8. Check the Hyrax openings ($E(r_x)$ and $W(r_y)$ against folded row commitments).
9. Check evaluation commitments for each PCS opening.

This verification algorithm is itself a deterministic program. Making $M$ a witness in step 2 means the R1CS construction (which currently reads the model graph to determine stage sizes, operator types, and variable counts) happens inside the recursive proof. The recursive prover $P_V$ runs the full verifier algorithm with $M$ in its witness, and the outer verifier $V'$ only checks that $P_V$'s execution was correct.

The key advantage over approaches that require redesigning the constraint system (pR1CS, HD-R1CS) is that **no changes to the inner proof system are needed**. The existing DAG-of-sumchecks pipeline, operator dispatch, eval-reduction, and batch opening all remain exactly as they are. The architecture hiding is achieved purely at the recursive composition layer.

### What the recursive verifier $V'$ needs

$V'$ receives:
- The commitment $\bar{M}$ to the model (a constant-size group element).
- The public inputs and outputs $x$.
- The recursive proof $\pi_V$.

$V'$ does *not* receive: the model $M$, the inner proof $\pi_M$, any sumcheck transcript, any committed polynomials, or any intermediate evaluation. All of these are private witnesses of $P_V$.

### Cost analysis

The overhead of the recursive layer is the cost of proving the verifier algorithm $V$ inside a SNARK:

- **Verifier circuit size.** The Jolt Atlas verifier performs field arithmetic (transcript hashing, sumcheck round checks, polynomial evaluation checks) and pairing checks (HyperKZG/Hyrax openings). The dominant cost is the pairing operations, which are expensive inside a SNARK circuit but are a fixed-size computation independent of model size.
- **Prover time.** The recursive prover $P_V$ must run the inner verifier $V$ and then prove its execution. Since $V$ is succinct (its runtime is polylogarithmic in the model size), the recursive proving cost is modest relative to the inner proving cost.
- **Proof size.** $\pi_V$ is a single SNARK proof (constant size), which replaces $\pi_M$ and all associated commitments. The final proof sent to $V'$ may actually be *smaller* than the non-recursive proof.

### Comparison with direct approaches

| | Recursive verification (zkARc) | Committed constraint matrices (pR1CS / HD-R1CS) |
|---|---|---|
| **Inner proof system changes** | None | Substantial: new constraint abstractions |
| **Prover overhead** | One extra recursive proof (verifier circuit) | Uniform operator encoding, padding, functional relation proof |
| **Verifier** | Checks a single recursive proof | Checks committed-circuit sumcheck |
| **What leaks** | Only $\bar{M}$ and I/O | Only size bound and I/O |
| **Implementation effort** | Moderate: need a SNARK-friendly verifier circuit | High: requires redesigning constraint generation |
| **Composability** | Natural: can recursively compose multiple proofs (model + policy + solver as in zkARc) | Each component needs its own committed-circuit adaptation |

The recursive approach is particularly attractive for Jolt Atlas because the inner proof system already defers all sumcheck claims to a small R1CS instance via NovaBlindFold. This means the verifier circuit that $P_V$ must prove is relatively compact.

### What the verifier learns

With recursive verification hiding both weights and architecture:

| Information | Visible? |
|---|---|
| That some committed model was executed correctly | Yes |
| Model inputs and outputs | Yes (unless input/output hiding is added) |
| Commitment to the model $\bar{M}$ (constant-size) | Yes |
| Architecture (operators, connectivity, shapes, hyperparameters) | **No** |
| Weight values | **No** |
| Intermediate activations | **No** |
| Inner proof structure (number of sumchecks, polynomial sizes) | **No** |

---

## Connection to NovaBlindFold

Jolt's BlindFold extension (as described in `.claude/hiding-model.md`) implements a similar weight-hiding pattern for R1CS-based proofs using Nova folding. The structural parallel is:

| Jolt BlindFold concept | Jolt Atlas equivalent |
|---|---|
| Weights become witness variables in the Nova instance W | Weights become committed `MultilinearPolynomial<F>` in `ModelSecretKey` |
| Verifier constructs R1CS from stage configs + public inputs only | Verifier constructs the IOP check from architecture metadata only |
| Real instance reconstructed from round commitments with u=1, E=0 | Prover registers opening claims for weight evaluations in `ProverOpeningAccumulator` |
| Hyrax opens W(r_y) against folded row commitments | HyperKZG opens weight MLE W(r) against published `ModelKey.weight_commitments` |
| Proof overhead: O(log P) group elements for GPT-2 | Expected proof overhead: same (one extra joint opening for all weight polynomials) |

The key difference is that Jolt Atlas uses a sumcheck-based IOP rather than Spartan over a folded Nova instance. The weight-hiding mechanism (commit before challenges; open at batch challenge point) translates directly because Jolt Atlas already uses exactly this pattern for its auxiliary witness polynomials.

For architecture hiding, NovaBlindFold plays a dual role. First, it already defers all sumcheck verification claims to a single small R1CS instance, which makes the inner verifier compact. Second, the recursive verification approach (proving the verifier with $M$ as witness) naturally composes with Nova's folding: the recursive prover $P_V$ can use the same NovaBlindFold machinery to prove the verifier circuit, keeping the recursive layer efficient.

---

## Related Work

The problem of hiding model parameters and architecture in verifiable ML inference is an active area of research. We survey the most relevant prior and concurrent work below.

**zkLLM.** Sun and Li \[[arXiv 2404.16109](https://arxiv.org/abs/2404.16109)\] present a zero-knowledge proof system specialised for large language models. They introduce *tlookup*, a parallelised lookup argument for non-arithmetic tensor operations, and *zkAttn*, a specialised proof for the attention mechanism. For LLMs with 13 billion parameters, zkLLM generates proofs in under 15 minutes with proof sizes under 200 kB while keeping model parameters private. Their weight-hiding approach uses polynomial commitments over the weight tensors, which is conceptually similar to Jolt Atlas's planned HyperKZG-based weight commitment scheme.

**zkPyTorch.** \[[ePrint 2025/535](https://eprint.iacr.org/2025/535)\] proposes a hierarchical optimising compiler that automatically generates zero-knowledge proofs from PyTorch models, verifying inference correctness while safeguarding model confidentiality. This compiler-based approach is complementary to Jolt Atlas's operator-level design.

**ZEN and zkCNN.** Earlier work on sumcheck-based proofs for neural networks, notably ZEN \[[ePrint 2021/87](https://eprint.iacr.org/2021/87)\] and zkCNN \[[ePrint 2021/673](https://eprint.iacr.org/2021/673)\], demonstrated the feasibility of GKR-style interactive proofs for CNNs. These systems focus on verifiability rather than model hiding, but their sumcheck machinery influenced the design of later weight-hiding schemes including our own.

**ZKML (EuroSys 2024).** Kang et al. \[[EuroSys 2024](https://ddkang.github.io/papers/2024/zkml-eurosys.pdf)\] present an optimising system for ML inference in zero-knowledge proofs, focusing on efficient constraint generation and proof composition for ONNX models. Their system-level optimisations (operator fusion, memory layout) are orthogonal to the cryptographic model-hiding techniques discussed here.

**Survey.** For a comprehensive overview of verifiable ML via zero-knowledge proofs, including verifiable training, testing, and inference, see the survey by Li et al. \[[arXiv 2502.18535](https://arxiv.org/abs/2502.18535)\].

---

## Implementation Path

### Weight hiding

The weight-hiding design maps to the `ModelKey`/`ModelSecretKey` plan in the roadmap (see [Roadmap](./roadmap.md)). The concrete steps are:

1. Add `weight_commitments: Vec<PCS::Commitment>` to a new `ModelKey` struct.
2. Strip `Tensor<i32>` data from `Constant` variants in the model passed to `AtlasSharedPreprocessing`.
3. In `ONNXProof::prove`, commit all weight polynomials before drawing any challenge, using the same HyperKZG path as `commit_witness_polynomials`.
4. Extend `CommittedPolynomial` (in `common`) with a `WeightPolynomial(node_idx)` variant and implement `WitnessGenerator` for it.
5. Modify each affected operator's `prove` method to register a weight opening claim in the `ProverOpeningAccumulator` instead of evaluating the weight directly.
6. Modify the corresponding `verify` method to accept the opening claim rather than reconstructing the evaluation from public data.
7. Extend `prove_reduced_openings` and `verify_reduced_openings` to include weight polynomials in the combined poly map.

Steps 3-7 follow patterns that already exist in the codebase: the quotient polynomials (`DivNodeQuotient`, `RsqrtNodeInv`) and lookup-address polynomials (`GatherRa`, `NodeOutputRaD`, etc.) already use exactly this commit-then-open flow.

### Architecture hiding (recursive verification)

Architecture hiding via the recursive verification approach from zkARc requires:

1. Implement the Jolt Atlas verifier as a SNARK-friendly circuit. The verifier already reduces to field arithmetic and pairing checks, so the main work is expressing the Fiat-Shamir transcript reconstruction and HyperKZG/Hyrax verification as R1CS constraints.
2. Build the recursive prover $P_V$ that runs the inner verifier $V$ with the model $M$ as a private witness and produces $\pi_V$.
3. Define the commitment scheme for $\bar{M} = \text{Com}(M)$, binding the model's architecture and weights into a single constant-size commitment that $V'$ can check against.
4. Implement $V'$, the recursive verifier, which checks $\pi_V$ using only $\bar{M}$ and the public I/O.

The key advantage of this approach is that steps 1-4 do not require any changes to the inner proof system. The existing DAG-of-sumchecks pipeline, operator dispatch, eval-reduction, and batch opening all remain unchanged.

An alternative approach using committed constraint matrices (pR1CS \[[ePrint 2025/2211](https://eprint.iacr.org/2025/2211)\], HD-R1CS \[[ePrint 2026/111](https://eprint.iacr.org/2026/111)\]) would require more fundamental restructuring of the verifier but could avoid the overhead of recursive composition.

---

## Do We Need NovaBlindFold for Input Zero-Knowledge?

A natural follow-up question is whether implementing NovaBlindFold is a prerequisite for any of the hiding properties above. The short answer is that **NovaBlindFold is not on the critical path for model hiding**, but it *is* necessary if you want a *formal* zero-knowledge guarantee on hidden inputs (as opposed to the heuristic "random evaluations are computationally useless" argument that the rest of this page relies on). The three properties have to be kept distinct:

| Property | What it hides | Mechanism |
|---|---|---|
| **Weight hiding** | Values of `Operator::Constant` tensors | Commit weight MLEs with HyperKZG before any FS challenge; replace direct $W$ evaluations with opening claims, batched into the existing `ProverOpeningAccumulator`. **No BlindFold needed.** |
| **Architecture hiding** | Operator types, shapes, DAG | Recursive verification à la zkARc, proving $V(M, \pi_M, x) = 1$ with $M$ as a private witness. NovaBlindFold isn't strictly required, but composes naturally because it already keeps the inner verifier compact. |
| **Input/output hiding** | Plaintext input/output tensors | Not part of "model hiding" as defined above; treated separately. See below. |

When this page (and the [Roadmap](./roadmap.md)) talks about "model hiding" it means weights ± architecture, and explicitly leaves inputs/outputs in the clear (see the `What Remains Public with Weight Hiding Only` table earlier on this page, and the equivalent table in [Security Considerations](../appendix/security.md)). Neither weight hiding nor architecture hiding requires BlindFold: the commit-then-open flow that weight hiding needs already exists for auxiliary witness polynomials (`DivNodeQuotient`, `GatherRa`, `NodeOutputRaD`, …), and architecture hiding is implemented one level up via the recursive composition layer.

### Where NovaBlindFold actually becomes necessary

If you genuinely want the *inputs* to be private from the verifier with a formal ZK guarantee, two things must happen:

1. **Move inputs out of `ModelExecutionIO`.** Today the input tensors are passed to `verify` in plaintext. To hide them, the input MLEs must become committed witness polynomials at preprocessing (mirroring the planned treatment of weights), and any operator that consumes an input must consume it via an opening claim rather than a direct tensor read.

2. **Blind the polynomial evaluations that depend on those inputs.** This is the part that needs BlindFold-style machinery, and the current system does not provide it. From [Security Considerations](../appendix/security.md):

   > **No Zero-Knowledge for Activation Values.** The current system does not implement BlindFold-style zero-knowledge for the intermediate activation trace. The sumcheck proofs leak information about polynomial evaluations at random points. In practice, this information is computationally useless (random linear combinations of tensor entries), but a formal ZK guarantee is not provided.

   The activation trace is a function of the inputs, so even after committing the input polynomial, sumcheck round messages and HyperKZG openings of activation-touching polynomials still leak field elements that depend on the inputs. They are heuristically "random-looking", and that is exactly the same heuristic argument the weight-hiding scheme on this page uses ("a single MLE evaluation at a uniformly random point is computationally indistinguishable from random under DL"). Without a blinding step, there is no formal proof that an adversary cannot recover information about the inputs.

   NovaBlindFold (or any equivalent, i.e., masking polynomials in sumcheck plus hiding commitments in the PCS) is the standard fix. Implementing it closes the activation-trace caveat as a side benefit.

### Decision matrix

Whether you need NovaBlindFold depends on your threat model:

| Threat model | NovaBlindFold required? | What to implement |
|---|---|---|
| Only the model is sensitive; the verifier may see inputs and outputs. | **No** | Weight hiding via committed weight MLEs; optionally recursive verification for architecture hiding. |
| Inputs must be hidden, but you accept the same heuristic argument used for weight hiding. | **No** (with caveat) | Move inputs into committed witness polynomials and add opening claims for input-touching operators. You inherit the "computationally useless leakage" caveat that already applies to activations. |
| Inputs must be hidden with a *formal* ZK guarantee. | **Yes** | NovaBlindFold (or an equivalent blinding scheme). Closes both the input-hiding gap and the existing activation-trace ZK gap. |

### A nicer alternative: defer input hiding to the recursive layer

There is one subtlety worth flagging. If architecture hiding is implemented via the recursive verification approach described above, the inner Jolt Atlas verifier is run as a private witness inside an outer SNARK. If the inputs are *also* passed as private witnesses to that outer proof (rather than as part of the public statement), then the outer SNARK's own zero-knowledge property gives input hiding *for free*, without needing BlindFold inside the inner Jolt Atlas pipeline.

This suggests a useful design point: rather than retrofit BlindFold into the inner system, **defer input hiding to the recursive composition layer**. The trade-off is:

- **Pro:** No invasive changes to the inner sumcheck / IOP machinery. Input hiding becomes a property of the recursive wrapper, which you would need anyway for architecture hiding.
- **Con:** Only available if you're already paying the recursive-verification cost for architecture hiding. If you only want input hiding without architecture hiding, retrofitting BlindFold (or a smaller masking scheme) into the inner system is the more direct path.
