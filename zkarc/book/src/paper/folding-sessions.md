# Folding sessions

The [proof aggregation](./proof-aggregation.md) page describes the two-tier
design as a property of the proof stack: same-session proofs fold via Nova,
cross-session proofs are discharged recursively. This page names the runtime
object that enforces the split, the **session**, and gives a concrete API.

A session is a proving-time scope that owns a set of BlindFold witnesses and
an in-progress folded `RelaxedR1CSInstance`. Components added to the same
session are folded directly. Components that finished in a previous session
are carried in as `RecursiveCredential`s and discharged by a verifier
subcircuit added to the current session's R1CS. The session ends with a
single Spartan decider proof.

## Why a runtime object

Nova folding requires the cross-term

$$
T = (A Z_1) \circ (B Z_2) + (A Z_2) \circ (B Z_1) - u_1 (C Z_2) - u_2 (C Z_1),
$$

which depends on **both witnesses simultaneously**. The witness-lifetime
constraint is therefore an API-level concern, not a property of any single
proof. Without an explicit object, it is easy for callers to discard a
witness too early (forcing recursion where folding would have sufficed) or
to keep witnesses alive across time horizons (defeating the whole point of
the setup-time / per-action split).

## What already exists

Both upstream codebases ship the same BlindFold subprotocol:

- `RelaxedR1CSInstance::fold(other, t_row_commitments, r)` in
  `jolt-core/src/subprotocols/blindfold/relaxed_r1cs.rs` and the
  byte-identical copy in `joltworks/src/subprotocols/blindfold/`.
- `compute_cross_term`, `sample_random_satisfying_pair` in
  `subprotocols/blindfold/folding.rs`.
- `OutputClaimConstraint` and `InputClaimConstraint` over
  `ValueSource::{Opening, Challenge, Constant}` in
  `subprotocols/blindfold/output_constraint.rs`.
- `StageConfig::initial_input` and `StageConfig::final_output` slots that
  bind one stage's output to the next stage's input inside a single prover
  run, in `subprotocols/blindfold/mod.rs`.

The Jolt `CLAUDE.md` already states that BlindFold uses Nova folding to
hide a single prover's witness. The session layer reuses that same fold
across components rather than across rounds of one prover.

What is missing today: an in-circuit verifier for a serialised BlindFold
proof (Tier 2), a PCS reconciliation between Dory and HyperKZG (the topic
of [proof aggregation](./proof-aggregation.md)), and the orchestration
object described below.

## Core abstractions

```text
   Component ───────────────►  BlindFold instance  ─┐
                               + witness            │ Tier 1: Nova fold
   Component ───────────────►  BlindFold instance  ─┤   in same Session
                               + witness            │
                                                    ▼
                               folded RelaxedR1CSInstance
                                                    │
   RecursiveCredential  ────►  verifier subcircuit ─┤ Tier 2: discharge
   (serialised proof,          constraints into     │   pre-existing proof
    no witness)                this session's R1CS  │
                                                    ▼
                               Spartan decider
                                                    │
                                                    ▼
                                       Receipt  (sent to verifier)
```

### Component

A component knows how to produce a `RelaxedR1CSInstance` plus its witness,
the public inputs that downstream components might want to bind to, and the
challenge values its constraints expect. The four concrete instances that
matter for zkARc are:

| Component | Backend | When |
|-----------|---------|------|
| $\pi_{\mathsf{SMTpolicy}}$ | Jolt Atlas | setup, once |
| $\pi_{\mathsf{NLplan}}$ (optional) | Jolt Atlas | per action |
| $\pi_{\mathsf{SMTplan}}$ | Jolt Atlas | per action |
| $\pi_{\mathsf{SMTsolver}}$ | Jolt | per action |

### Session

```rust
let mut session = Session::new(transcript, params);
let av_handle  = session.add(plan_component, plan_witness)?;
let smt_handle = session.add(smt_component,  smt_witness)?;
session.bind(av_handle.output("C_SMT"), smt_handle.input("C_SMT"))?;
session.attach(policy_credential, "P_SMT", smt_handle.input("P_SMT"))?;
let receipt = session.finalize()?;
```

`Session::add` calls into the component's BlindFold prover, gets a
`RelaxedR1CSInstance`, and folds it into the session's running instance via
`RelaxedR1CSInstance::fold`. Cross-component bindings are expressed as
`OutputClaimConstraint`s emitted into the next component's `initial_input`
slot.

`Session::attach` consumes a `RecursiveCredential` (Tier 2). The credential
is not folded; instead, the session's R1CS gains a verifier subcircuit
that asserts validity of the credential's serialised BlindFold proof. The
credential's exposed commitments (for example, $\mathsf{Com}(P_{\mathsf{SMT}})$)
become public inputs that downstream `OutputClaimConstraint`s can reference.

### RecursiveCredential

```rust
pub struct RecursiveCredential {
    pub backend:     Backend,                 // Jolt or JoltAtlas
    pub pcs:         PcsKind,                 // HyperKZG or Dory
    pub folded_blob: Vec<u8>,                 // serialised RelaxedR1CSInstance
    pub decider:     DeciderProof,            // Spartan over the folded instance
    pub exposed:     Vec<NamedCommitment>,    // e.g. ("P_SMT", Com(P_SMT))
}
```

The credential is the persisted form of a previous session's receipt. The
new session that consumes it does not need the original witness, only the
proof and the named commitments that the previous session chose to expose.

### Receipt

```rust
pub struct Receipt {
    pub folded:               FoldedInstance,        // Tier 1 result
    pub decider:              DeciderProof,          // settles the folded instance
    pub public_outputs:       Vec<NamedCommitment>,  // commitments shown to verifier
    pub verified_credentials: Vec<VerifiedClaim>,    // Tier 2 anchors
    pub component_roles:      Vec<ProofRole>,        // for human-readable reporting
}
```

`receipt.verify()` runs Spartan over `folded`, verifies each PCS opening,
and rebinds the named commitments to the application-level claim, "policy
$P$ was checked against action $A$ and produced verdict $b$".

## Mapping the zkARc proofs onto the API

```rust
// Setup time, offline, separate process or different day.
let mut setup = Session::new(...);
let policy = setup.add(policy_component, policy_witness)?;
setup.expose(policy.output("P_SMT"))?;       // commit Com(P_SMT)
let policy_credential = setup.finalize()?.into_credential(ProofRole::SmtPolicy);
// Persist policy_credential. Just bytes, no witnesses.

// Action time, per action, hot path.
let mut act = Session::new(...);
let nlplan = act.add(nlplan_component, nlplan_witness)?;     // optional
let plan   = act.add(plan_component,   plan_witness)?;
let smt    = act.add(smt_component,    smt_witness)?;
act.bind(nlplan.output("A_NL"), plan.input("A_NL"))?;
act.bind(plan.output("C_SMT"),  smt.input("C_SMT"))?;
act.attach(policy_credential, "P_SMT", smt.input("P_SMT"))?; // Tier 2
let receipt = act.finalize()?;                               // single Spartan
```

The verifier sees one `Receipt` and learns three facts:

1. The committed policy $\bar{P}$ has a Tier-2-verified
   $\pi_{\mathsf{SMTpolicy}}$.
2. The per-action proofs for $\pi_{\mathsf{NLplan}}$, $\pi_{\mathsf{SMTplan}}$,
   $\pi_{\mathsf{SMTsolver}}$ were Tier-1-folded together.
3. The verdict $b$ on $(P_{\mathsf{SMT}}, C_{\mathsf{SMT}})$ is bound to the
   same $\bar{P}$.

## PCS reconciliation

The session API is parameterised over a `BlindFoldBackend` trait so the
choice of PCS-reconciliation strategy from the
[proof aggregation](./proof-aggregation.md) page is local to the backend
implementation:

- `Unified<HyperKzg>`: easiest now, since HyperKZG already exists in
  `jolt-core/src/poly/commitment/hyperkzg.rs`. Loses Dory's transparent setup.
- `Padded<Dory, HyperKzg>`: pads each verifier subcircuit to a common shape
  with selector wires. No folding-scheme change.
- `Ccs<HyperNova>`: native fold of different shapes; biggest engineering
  lift.

The default in the prototype is `Unified<HyperKzg>` for the same reason: both
repos already have HyperKZG plumbing.

## Prototype

A prototype implementation of the session API lives at
`code/zkarc-folding-sessions/`. It is a small Rust workspace with no
arkworks dependency, exercising the orchestration with a `MockBackend`:

- `crates/sessions/src/`: `Session`, `Component`, `RecursiveCredential`,
  `Receipt`, `BlindFoldBackend` trait, plus a `Transcript` shim.
- `crates/sessions/examples/zkarc_pipeline.rs`: end-to-end mock walk-through
  matching the snippet above (setup session produces a policy credential;
  action session folds `NLplan`, `SMTplan`, `SMTsolver` and attaches the
  policy credential).
- `crates/sessions/tests/orchestration.rs`: five tests covering Tier 1
  folding, Tier 2 attachment, malformed bindings, empty session, and
  cross-backend rejection.

The prototype's purpose is to fix the API surface so a follow-up
integration step can replace the `MockBackend` with adapters over
`jolt_core::subprotocols::blindfold::BlindFoldProver` and
`joltworks::subprotocols::blindfold::BlindFoldProver` without churning
session-level code.

## Open work

1. Replace `MockBackend` with adapters over the real BlindFold provers in
   each upstream repo.
2. Implement the in-circuit verifier subcircuit for Tier 2. Smallest first
   cut: a HyperKZG pairing-check subcircuit over BN254, hard-coded to
   verify a single Atlas $\pi_{\mathsf{SMTpolicy}}$, generalised after.
3. Run an end-to-end measurement on a 50-rule policy plus a small ONNX
   answer verifier plus a Rust SMT solver, and report Spartan-only verifier
   time.
