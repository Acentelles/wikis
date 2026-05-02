# Prototype: folding sessions in Rust

A working prototype of the [folding-sessions](./folding-sessions.md) API
lives at `code/zkarc-folding-sessions/`. It is not a full prover. It is a
structural prototype that fixes the API surface and validates the
mathematical core (Nova fold over relaxed R1CS) end-to-end against the
real `joltworks::subprotocols::blindfold` types. This page documents
what the prototype does, how it is wired, and what remains.

## What is in the workspace

```text
code/zkarc-folding-sessions/
├─ Cargo.toml              workspace + [patch.crates-io] for forked arkworks
├─ rust-toolchain.toml     pinned to 1.88 to match jolt-atlas-main
├─ Cargo.lock              copied verbatim from jolt-atlas-main
├─ DESIGN.md               long-form sketch (the source of the wiki page)
└─ crates/
    ├─ sessions/                core orchestration crate (no arkworks)
    │   └─ src/{backend,session,component,credential,receipt,
    │           claim,error,transcript,lib}.rs
    └─ sessions-jolt-atlas/     real adapter (depends on joltworks)
        └─ src/{lib,pad}.rs
```

The two-crate split is deliberate. The core `sessions` crate has no
crypto dependencies and compiles in seconds; it owns the trait, the
session state machine, and the mock backend used in unit tests. The
adapter crate is the heavy one: it depends on `joltworks` (with the
forked arkworks toolchain) and implements the trait against real
`RelaxedR1CSInstance<F, C>` and `RelaxedR1CSWitness<F>`.

## Status of the five layers

The proposed [Session API](./folding-sessions.md) has five layers. Their
current status:

| Layer | Status | Notes |
|---|---|---|
| `BlindFoldBackend` trait | ✅ done | Three associated types: `Instance`, `FoldedInstance`, `Decider`. |
| `Component<I>`, `ClaimHandle`, `Session::add` | ✅ done | Generic over the backend's `Instance`. |
| `Session::bind` | partial | Records the link; does not yet emit an `OutputClaimConstraint`. |
| `Session::attach` (Tier 2) | partial | Records the credential and binds it to a local input slot, but the in-circuit verifier subcircuit is not yet implemented. |
| `Session::finalize` + `Receipt` | done (Atlas backend) | Builds a receipt with the folded relaxed R1CS instance and a real `AtlasDecider`: outer Spartan sumcheck, inner sumcheck, and Hyrax openings of `W` and `E`. A matching verifier on the same backend accepts the proof end-to-end (`prover_decider_verifies_end_to_end` test) and rejects tampered final claims (`verifier_rejects_tampered_decider`). |

What works end-to-end against real arkworks: the Tier-1 fold itself.
That is, given two `RelaxedR1CSInstance<F, C>` instances of the same
shape, the prototype computes the Nova cross-term, commits its rows via
Pedersen, derives a Fiat-Shamir challenge, and folds both the instance
and the witness. The folded triple `(instance, witness, u)` satisfies
$(A Z) \circ (B Z) = u (C Z) + E$, which is verified in the tests via
`RelaxedR1CSWitness::check_satisfaction`.

## The real adapter

`crates/sessions-jolt-atlas/src/lib.rs::JoltAtlasBackend<F, C>` is the
real implementation of `BlindFoldBackend`. It holds:

- `r1cs: Arc<VerifierR1CS<F>>`: the shared verifier R1CS that all
  instances in the session must have been produced against.
- `gens: Arc<PedersenGenerators<C>>`: the shared row-commitment
  generators.
- `seed: [u8; 32]`: deterministic seed for cross-term blinding RNGs.

The `fold` method does the four things the paper's BlindFold appendix
describes for the generic NovaBlindFold construction:

1. `compute_cross_term(&r1cs, z1, u1, z2, u2)` produces `T`.
2. `commit_cross_term_rows(&gens, &T, R_E, C_E, &mut rng)` produces
   `(t_row_commits, t_row_blindings)`.
3. A Fiat-Shamir challenge `r` is squeezed from the session's
   transcript and reduced into the field via `ChaCha20Rng`.
4. `instance.fold(&other_instance, &t_row_commits, r)` folds the
   commitments; `witness.fold(&other_witness, &T, &t_row_blindings, r)`
   folds the witness.

The `decide` method runs the full Spartan + inner-sumcheck flow against
a `Blake2bTranscript` carried inside `AtlasFolded`:

1. Outer Spartan sumcheck via `BlindFoldSpartanProver`: $\log m$ rounds,
   each producing a degree-3 round polynomial; the verifier-readable
   coefficients are appended to the transcript before the next
   challenge is squeezed.
2. Final claims `(az_r, bz_r, cz_r)` are appended; Fiat-Shamir
   challenges $(r_a, r_b, r_c)$ are squeezed.
3. Inner sumcheck via `BlindFoldInnerSumcheckProver`: $\log |W|$ rounds,
   degree 2.
4. Hyrax-style openings of the folded witness `W` (at $r_y$) and folded
   error vector `E` (at $r_x$) over the row-grid layout.

The result is an `AtlasDecider<F, C>` carrying the two sumcheck proof
vectors, the three final claims, and the two Hyrax openings. The
`JoltAtlasBackend::verify` method mirrors the prover transcript path
exactly: re-squeeze $\tau$, replay each round's sum check, re-derive
$(r_a, r_b, r_c)$, replay the inner sumcheck, verify the Hyrax E and W
openings against the folded instance's row commitments, and finally
check the outer-claim relation
$\widetilde{Az}(r_x)\,\widetilde{Bz}(r_x) = u\,\widetilde{Cz}(r_x) + \widetilde{E}(r_x)$
and the inner-claim relation $\text{inner\_claim} = L_w(r_y) \cdot W(r_y)$.

## The padding helper

`crates/sessions-jolt-atlas/src/pad.rs::unified_r1cs` implements the
shape-reconciliation strategy described in
[Proof aggregation](./proof-aggregation.md). The function takes a list
of `UnionPart<F>`, each carrying one component's `StageConfig`s,
`OutputClaimConstraint`s, and `BakedPublicInputs`, and concatenates them
into a single `VerifierR1CS<F>`. Every component then produces its
instances against this shared shape, sidestepping Nova's same-shape
requirement upfront.

This is structurally simpler than block-diagonal embedding (which
would lift already-built different-shape instances into a common shape
after the fact) and is sound because the per-action proof stack
(Provenance + AV + SMT) is fixed at preprocessing time. The same
strategy generalises to any session whose component set is known at
construction.

## Tests that pass today

```bash
cargo test -p zkarc-sessions             # 5 orchestration tests, mock backend
cargo test -p zkarc-sessions-jolt-atlas  # 3 tests, real arkworks types
```

Mock-backend orchestration tests (`crates/sessions/tests/orchestration.rs`):

- Tier-1 per-action session folds three components.
- Tier-2 credential attaches into an action session.
- Binding to an unknown input slot is rejected.
- Empty sessions cannot finalise.
- Cross-backend folds are rejected after the first component.

Real-adapter tests (`crates/sessions-jolt-atlas/src/{lib,pad}.rs`):

- Two same-shape `RelaxedR1CSInstance`s, sampled by
  `sample_random_satisfying_pair`, fold and the result still satisfies
  the relaxed R1CS relation.
- The same with three folds in a row.
- Two parts with different `StageConfig`s are reconciled by
  `unified_r1cs` into a common shape, three folds against that shape,
  satisfaction holds.
- `decide` produces a real Spartan + inner-sumcheck proof of the right
  shape (one outer round per `log_2(num_constraints)`, one inner round
  per `log_2(|W|)`) over the session-folded relaxed R1CS instance.
- The decider is deterministic: same inputs and same seed produce the
  same proof, byte-for-byte.
- `prover_decider_verifies_end_to_end`: a three-component fold +
  decide + verify accepts, mirroring what a real verifier sees.
- `verifier_rejects_tampered_decider`: flipping `az_r` makes the
  verifier return `OuterClaimMismatch` (or one of the related Spartan
  errors).

## Dependency wiring (the part that took the most time)

Pulling in `joltworks` from a sibling repo as a path dependency required
matching its forked arkworks patches in our workspace `Cargo.toml`:

```toml
[patch.crates-io]
ark-ff        = { git = "https://github.com/a16z/arkworks-algebra", branch = "dev/twist-shout" }
ark-ec        = { git = "...", branch = "dev/twist-shout" }
ark-poly      = { git = "...", branch = "dev/twist-shout" }
ark-serialize = { git = "...", branch = "dev/twist-shout" }
ark-bn254     = { git = "...", branch = "dev/twist-shout" }
allocative        = { git = "https://github.com/facebookexperimental/allocative", rev = "85b773d8..." }
allocative_derive = { git = "...", rev = "85b773d8..." }
```

Three failure modes were ironed out:

1. *`ark-poly` failing to compile* because the registry version uses
   crates.io `ark-serialize` while we patched `ark-ff` to the fork.
   Fixed by patching all of `ark-{ec,poly,serialize,bn254}`.
2. *`tract-onnx` requiring rustc 1.91* because it was resolved to a
   newer version than `jolt-atlas-main`'s lockfile. Fixed by copying
   `jolt-atlas-main/Cargo.lock` verbatim and pinning the toolchain to
   1.88.
3. *Two `allocative` versions co-existing* (registry + git) because the
   forked arkworks types only implement the git version's trait. Fixed
   by patching the registry version to point at the same git rev.

The `joltworks` features `["test-feature", "zk"]` are required to
expose `VerifierR1CSBuilder::new` and the `subprotocols::blindfold`
module respectively.

## What is not yet done

These are the open items from the prototype's perspective. Each maps
to a concrete file or set of files.

- **`OutputClaimConstraint` emission** in `Session::bind` and
  `Session::attach` (`crates/sessions/src/session.rs`). At present the
  binding is recorded but does not propagate into the next component's
  `StageConfig::initial_input`.
- **In-circuit BlindFold verifier subcircuit** for Tier 2. Smallest
  first cut: a HyperKZG pairing-check subcircuit hard-coded for
  $\pi_{\mathsf{PMC}}$, following
  [Efficient recursion for aggregation](./efficient-recursion.md).
- **Block-diagonal padding** (the alternative to `unified_r1cs` that
  works on already-built instances). Stub at
  `crates/sessions-jolt-atlas/src/pad.rs`.
- **Vertical-folding support** (segmented Jolt runs that the
  Proving-budget paragraph in the paper now claims). Requires either
  repeated `add` calls of the same role or an explicit
  `add_segments` helper.

## How this maps to the paper

| Paper claim | Prototype fact |
|---|---|
| Tier-1 same-session folding via Nova NIFS | Implemented and tested over real arkworks. |
| Tier-2 cross-session via recursive verification | API recorded; in-circuit verifier subcircuit not yet built. |
| Single Spartan decider over the folded instance | Implemented; outer + inner sumcheck + Hyrax openings produced and verified end-to-end. |
| Cross-component bindings via `OutputClaimConstraint` | API records; emission not yet wired. |
| PCS reconciliation (unify on HyperKZG) | Reflected in the adapter's `pcs_kind() == HyperKzg`; the unified-R1CS path in `pad.rs` is the implementation. |
| The session names the runtime that owns BlindFold witnesses | The Rust types make this explicit: `Session<'b, B>` owns `Vec<Component<B::Instance>>` and the running `B::FoldedInstance`. |

## Pointers

- The DESIGN.md inside the workspace has the long-form sketch, written
  before the real adapter was wired; this wiki page is the up-to-date
  status.
- The [Folding sessions](./folding-sessions.md) page is the
  paper-aligned story; this page is its implementation companion.
- The [Efficient recursion](./efficient-recursion.md) page describes
  the technique that will populate the Tier-2 verifier subcircuit.
