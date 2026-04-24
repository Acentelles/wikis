# Proof Aggregation

Jolt Atlas proofs do not exist in isolation. In systems like zkARc, a single end-to-end claim may require combining proofs from different proving systems: Jolt Atlas proofs for ML inference, Jolt proofs for RISC-V program execution (e.g., an SMT solver), and potentially other SNARKs. This page describes the problem, the approaches, and the path forward.

---

## Motivation

The zkARc architecture produces a modular stack of proofs:

| Proof | System | What it proves |
|---|---|---|
| $\pi_{\mathsf{NLplan}}$ | Jolt Atlas | Planner model transformed prompt into action plan |
| $\pi_{\mathsf{SMTplan}}$ | Jolt Atlas | Answer-verifier model extracted logical claims from action plan |
| $\pi_{\mathsf{SMTpolicy}}$ | Jolt Atlas | Policy model compiled natural-language policy into formal SMT-LIB |
| $\pi_{\mathsf{SMTsolver}}$ | Jolt | SMT solver correctly checked claims against policy |

Sending four separate proofs to a verifier is impractical for high-throughput agentic commerce. The goal is to compress this stack into a **single succinct receipt** that attests to the entire chain from natural language to formal verification.

---

## The PCS Compatibility Problem

Jolt and Jolt Atlas use different polynomial commitment schemes:

| | Jolt (zkVM) | Jolt Atlas (zkML) |
|---|---|---|
| **PCS** | Dory (Pedersen-based MSM) | HyperKZG (pairing-based KZG) |
| **Setup** | Transparent (no trusted setup) | Trusted setup (SRS with toxic waste) |
| **Verification** | Inner-product argument | Pairing check |
| **Proof size** | $O(\sqrt{n})$ group elements | $O(\log n)$ group elements |
| **On-chain cost** | Higher (no pairings) | Lower (pairing-friendly) |

Both systems share the same underlying architecture: a DAG of sumcheck instances whose verification claims are deferred to a single R1CS instance via NovaBlindFold. The difference is only in the final opening proof, where Dory uses Pedersen commitments with an inner-product argument and HyperKZG uses KZG commitments with a pairing check.

This PCS divergence means proofs from the two systems cannot be directly folded or batched at the polynomial commitment level. Their R1CS instances have different shapes because the PCS verification circuits are different.

---

## Aggregation Strategies

### Strategy 1: Unify on a single PCS

The simplest approach is to make both systems use the same PCS. Two options:

**Option A: HyperKZG everywhere.** Replace Dory in Jolt with HyperKZG. This gives both systems identical proof structures, enabling direct folding at the R1CS level.

- *Advantage*: No aggregation machinery needed; proofs are natively compatible.
- *Advantage*: Smaller proofs and cheaper on-chain verification (pairings are EVM-friendly).
- *Cost*: Requires a trusted setup (SRS generation). Jolt currently avoids this.
- *Cost*: Jolt's Dory-specific optimizations (row-based Pedersen commitments, streaming) would need to be reworked.
- *Feasibility*: High. Jolt Atlas already has a production HyperKZG implementation. The Jolt Atlas whitepaper notes that "when a CPU trace is used to verify ONNX execution, Dory is used," confirming both PCS paths exist in the codebase.

**Option B: Dory everywhere.** Replace HyperKZG in Jolt Atlas with Dory.

- *Advantage*: Transparent setup.
- *Cost*: Larger proofs ($O(\sqrt{n})$ vs $O(\log n)$), more expensive on-chain verification (no pairings).
- *Feasibility*: Possible but moves in the wrong direction for on-chain use cases.

**Recommendation:** Option A (HyperKZG everywhere) is the pragmatic choice for zkARc, where on-chain verification cost is a primary concern and trusted setup is acceptable.

### Strategy 2: Recursive verification

Instead of unifying the PCS, prove each system's verifier inside a common SNARK:

```
π_atlas  ──→  P_agg proves: V_atlas(π_atlas) = accept
π_jolt   ──→  P_agg proves: V_jolt(π_jolt)   = accept
                         ↓
                    single π_agg
                         ↓
                   V_agg checks π_agg
```

The aggregation prover $P_{\mathsf{agg}}$ runs both verifiers as witness computations inside a single circuit and produces one proof $\pi_{\mathsf{agg}}$. The PCS difference is absorbed: $V_{\mathsf{atlas}}$'s pairing check and $V_{\mathsf{jolt}}$'s inner-product check both become R1CS constraints in the aggregation circuit.

- *Advantage*: No changes to either inner proving system.
- *Advantage*: Naturally extends to more proof types (SMT solver certificates, training proofs).
- *Cost*: Pairing operations inside a SNARK circuit are expensive (~tens of millions of R1CS constraints for a single BN254 pairing). This dominates the aggregation prover time.
- *Cost*: The aggregation circuit must be compiled and maintained separately.

This is the approach described in the [model hiding](./model-hiding.md) section for hiding the model, and it generalizes naturally to aggregation.

### Strategy 3: Folding at the R1CS level

Both Jolt and Jolt Atlas defer all sumcheck verification to a single R1CS instance via NovaBlindFold. The key insight is that the R1CS instances from both systems can potentially be **folded together** using Nova's folding scheme, even if the original PCS differs, provided the R1CS structure is made compatible.

This is the approach used by PlasmaBlind \[[ePrint 2026/634](https://eprint.iacr.org/2026/634)\], which links two different verification tasks (user balance proofs and Merkle tree state updates) through their shared inputs using Nova folding, avoiding the overhead of full recursive proof composition.

Applied to Jolt + Jolt Atlas:

1. Both systems produce a BlindFold R1CS instance encoding their respective sumcheck verifier checks.
2. The PCS-specific verification (Dory inner-product or HyperKZG pairing) is the only part that differs between the two R1CS circuits.
3. A folding step combines the two R1CS instances into one, using Nova's NIFS protocol.
4. A single Spartan proof over the folded instance settles all claims.

The challenge is that Nova folding requires both instances to have the **same R1CS shape** (same constraint matrices $A$, $B$, $C$). If the Dory and HyperKZG verification circuits have different shapes, they cannot be directly folded. Two approaches to resolve this:

**Padding to a uniform circuit.** Pad both verification circuits to the same shape. This wastes constraints but enables direct folding.

**CCS generalization.** Use HyperNova \[[ePrint 2023/573](https://eprint.iacr.org/2023/573)\], which generalizes Nova from R1CS to CCS (Customizable Constraint Systems). CCS can fold instances of different shapes by encoding them in a common format. This avoids padding overhead but requires migrating from Nova to HyperNova.

### Strategy 4: Deferred PCS aggregation

A hybrid approach that keeps the inner proofs separate but aggregates only the PCS openings:

1. Each system (Jolt, Jolt Atlas) produces its proof normally with its native PCS.
2. An aggregation layer collects all polynomial evaluation claims from both proofs.
3. A single batch opening proof settles all claims, using a PCS that can handle both Dory and HyperKZG commitments.

This requires a "multi-PCS" batch opening protocol, which does not currently exist in the literature as a standard construction. It is the most elegant approach in theory but the least mature in practice.

---

## Recommended Path

For the zkARc use case, we recommend a phased approach:

### Phase 1: Unify PCS (near-term)

Add HyperKZG as an alternative PCS backend in Jolt. This is feasible because:
- The HyperKZG implementation already exists in `joltworks/src/poly/commitment/hyperkzg/`.
- Both systems already use Spartan + NovaBlindFold for their R1CS layer.
- The Jolt Atlas whitepaper confirms both PCS paths coexist in the codebase.

With a shared PCS, proofs from both systems have identical structure and can be directly batched or folded.

### Phase 2: Fold at the R1CS level (medium-term)

With unified PCS, leverage Nova folding to combine the BlindFold R1CS instances from multiple proofs (multiple Jolt Atlas inference proofs + Jolt solver proof) into a single folded instance. This produces a constant-size aggregated proof regardless of the number of component proofs.

The PlasmaBlind approach \[[ePrint 2026/634](https://eprint.iacr.org/2026/634)\] demonstrates this pattern: linking different verification tasks via shared inputs under Nova folding, with sub-100ms client-side proving overhead per fold.

### Phase 3: Recursive composition for heterogeneous proofs (longer-term)

For proof types that cannot be unified under a single PCS (e.g., third-party SMT solver certificates, Groth16 proofs from other systems), fall back to recursive verification: prove the foreign verifier inside a SNARK circuit and fold the resulting R1CS instance into the aggregated proof.

---

## Aggregation Architecture

The following diagram shows the full aggregation pipeline for a zkARc proof stack:

```
  Jolt Atlas (HyperKZG)              Jolt (HyperKZG, after Phase 1)
  ┌──────────────────┐               ┌──────────────────┐
  │ π_NLplan         │               │ π_SMTsolver      │
  │ π_SMTplan        │               │                  │
  │ π_SMTpolicy      │               │                  │
  └────────┬─────────┘               └────────┬─────────┘
           │                                   │
           │  BlindFold R1CS instances         │  BlindFold R1CS instance
           ▼                                   ▼
  ┌─────────────────────────────────────────────────────┐
  │              Nova Folding (Phase 2)                 │
  │                                                     │
  │  Fold all R1CS instances into a single              │
  │  folded relaxed R1CS instance (U*, W*)              │
  └────────────────────┬────────────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────────────┐
  │           Spartan Decider (single proof)            │
  │                                                     │
  │  Produces π_agg: constant-size proof                │
  │  that all component proofs verified correctly       │
  └────────────────────┬────────────────────────────────┘
                       │
                       ▼
                  V_agg checks π_agg
                  using only public I/O
                  and model commitments
```

---

## What the Aggregated Verifier Checks

The aggregated verifier $V_{\mathsf{agg}}$ receives:
- The aggregated proof $\pi_{\mathsf{agg}}$ (constant size).
- Public inputs/outputs for each component: model I/O for inference proofs, solver verdict for the SMT proof.
- Commitments to hidden values (model commitments $\bar{M}$, policy commitments, etc.).

$V_{\mathsf{agg}}$ does not see: any model weights, any intermediate activations, any sumcheck transcripts, or any individual component proof. It verifies a single Spartan proof over the folded R1CS instance.

---

## Two-tier aggregation: folding within sessions, recursion across sessions

The phased approach above implicitly assumes all proofs are produced in the same session. In practice, the zkARc proof stack spans two time horizons:

- **Setup-time proofs.** $\pi_{\mathsf{SMTpolicy}}$ is produced once when the policy is authored or updated, potentially days before any action is checked. Only the proof (not the witness) is stored.
- **Per-action proofs.** $\pi_{\mathsf{SMTplan}}$, $\pi_{\mathsf{SMTsolver}}$, and optionally $\pi_{\mathsf{NLplan}}$ are produced in real time with witnesses in memory.

Nova folding requires both witnesses to be available simultaneously (to compute the cross-term $T$). If the policy proof was generated a week ago and the witness discarded, pure folding cannot incorporate it.

**Resolution: two tiers.**

- **Tier 1 (folding).** Per-action proofs are produced in the same session with witnesses in memory. They fold directly via Nova NIFS into one `RelaxedR1CSInstance`, settled by a single Spartan decider. This is the fast path.
- **Tier 2 (recursion).** The setup-time proof $\pi_{\mathsf{SMTpolicy}}$ is verified *inside* the Tier 1 circuit: the per-action prover takes the serialised policy proof as a public input and checks its validity, binding the output commitment $\bar{P}_{\mathsf{SMT}}$ to the policy input of $\pi_{\mathsf{SMTsolver}}$. Since the Jolt Atlas verifier is already compressed to a single R1CS check + PCS opening via BlindFold, this recursive subcircuit is compact.

**Result:** the aggregated verifier checks one proof, sees the commitments and the verdict, and is done. The policy proof acts as a reusable credential that the per-action prover carries and verifies.

---

## Proven-SLM translation correctness

A key question for zkARc is how to guarantee that the translation model (the SLM that converts natural-language actions to SMT claims) did its job correctly. Running $k$ translators and checking verdict agreement is not enough: without a ZKP, no translator is proven to have run.

The design adopted in the paper:

1. **Prove the SLM ran.** The local SLM is small enough for Jolt Atlas. $\pi_{\mathsf{SMTplan}}$ attests that a committed model executed on the action text and produced a specific translation. This is the binding anchor.
2. **Check agreement with a proprietary model.** A larger proprietary translator (e.g., the ARc production model) runs on trusted infrastructure and produces its own verdict. Inside the ZK circuit, a scalar equality check verifies that the proven SLM's solver verdict matches the proprietary verdict (supplied as a private witness).

The residual threat is narrow: a compromise of the proprietary infrastructure could inject a false agreement. But the guarantees are strictly better than either approach alone.

---

## Checker-based solver proofs

The solver proof $\pi_{\mathsf{SMTsolver}}$ has two instantiation options:

1. **Full solver in Jolt.** Oaksive (Rust SMT solver) compiled to RISC-V runs inside Jolt. Up to 50 rules within a ~2s proving budget. Provides solver-execution binding (the verifier knows *which* solver ran).
2. **Checker-based.** The solver runs natively and emits a certificate. A small checker (LRAT for SAT, Alethe for SMT) runs inside the zkVM. The ZK proof attests "there exists a certificate that passes this check." The certificate is a private witness (important for policy hiding). Cost scales with checker complexity (near-linear), not solver complexity (exponential worst-case).

The checker-based approach sidesteps the SMT proof-format standardisation debate: the ZK proof *is* the portable, succinct proof format.

---

## Execution-binding middleware

The paper introduces an enforcement-layer middleware that closes the gap between "the guardrail was checked" and "the checked action is the one that executed":

1. The agent generates a plan and obtains a zkARc proof that it passes the policy.
2. The middleware intercepts every outbound action and requires a valid proof before dispatch.
3. The counterparty's system rejects any action that arrives without a valid proof.

The middleware itself does not need to be trusted, because the proof is the trust anchor. The remaining gap (between action-as-received and action-that-executes-downstream) is a distributed-systems problem rather than a cryptographic one.

---

## Hidden-policy B2B trust

In hidden-policy mode, $P_{\mathsf{NL}}$ and $P_{\mathsf{SMT}}$ are private witnesses committed as $\bar{P}$. The counterparty trusts the proof but cannot inspect the policy. Three levels of trust address this:

1. **Auditor-mediated.** A third party inspects the policy, verifies it meets a standard, and signs $\bar{P}$.
2. **Meta-property proofs.** The policy owner proves in ZK that the committed policy satisfies structural properties ("contains a rule about X") without revealing the full rule set.
3. **Reputation-backed.** The counterparty trusts reputation for policy *content* but gets cryptographic assurance that the policy was *executed correctly*.

---

## Open Questions

1. **R1CS shape compatibility.** When folding Jolt and Jolt Atlas BlindFold instances, do the R1CS shapes match? If not, what is the padding overhead? A CCS/HyperNova approach would avoid this but requires migration.

2. **Shared SRS.** With HyperKZG in both systems, can they share a single SRS, or do the different maximum polynomial sizes require separate setups? Sharing an SRS simplifies deployment.

3. **Streaming aggregation.** Can proofs be folded incrementally as they arrive (e.g., fold $\pi_{\mathsf{NLplan}}$ first, then fold in $\pi_{\mathsf{SMTplan}}$ when ready), or must all proofs be available simultaneously? The two-tier design partially answers this: Tier 1 proofs must be simultaneous, but Tier 2 (policy proof) is pre-computed and only verified.

4. **Cross-proof data flow.** In zkARc, the output of one proof is the input to the next (e.g., the action plan output by the planner is the input to the answer verifier). How is this linkage enforced in the aggregated proof? Committed outputs from one component must match committed inputs of the next.

5. **Checker format maturity.** LRAT checkers are well-established for SAT. Alethe for SMT is emerging. Is the QF_NRIA fragment used by ARc covered by existing Alethe checkers, or does it require a custom checker?

---

## References

- PlasmaBlind: Daix-Moreux and Zhang, ["PlasmaBlind: A Private Layer 2 With Instant Client-Side Proving"](https://eprint.iacr.org/2026/634), ePrint 2026/634.
- PlasmaFold: Daix-Moreux et al., ["PlasmaFold"](https://eprint.iacr.org/2025/1300), ePrint 2025/1300.
- HyperNova: Setty, Thaler, Wahby, ["HyperNova: Recursive Arguments for Customizable Circuits"](https://eprint.iacr.org/2023/573), ePrint 2023/573.
- MicroNova: Setty and Thaler, ["MicroNova: Folding-scheme-based Recursive SNARKs without Forking"](https://eprint.iacr.org/2024/2099), ePrint 2024/2099.
- zkARc: Bayless et al., "A Neurosymbolic Approach to Natural Language Formalization and Verification," arXiv:2511.09008, 2025.
- Jolt Atlas: Benno, Centelles, Douchet, Gibran, "Jolt Atlas: Verifiable Inference via Lookup Arguments in Zero Knowledge," arXiv:2602.17452, 2026.
