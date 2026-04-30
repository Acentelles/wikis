# Proof aggregation

## The problem

The zkARc proof stack produces up to four proofs from two different proving
systems (Jolt Atlas for ML inference, Jolt for RISC-V/solver). Sending four
separate proofs is impractical for high-throughput agentic commerce. The goal
is a single succinct receipt.

## PCS compatibility

Jolt uses Dory (Pedersen-based, transparent setup, $O(\sqrt{n})$ proofs).
Jolt Atlas uses HyperKZG (pairing-based, trusted setup, $O(\log n)$ proofs).
The PCS difference means the BlindFold R1CS instances from each system have
different shapes, blocking direct Nova folding.

**Recommended resolution:** unify on HyperKZG. The implementation already
exists in both codebases. With a shared PCS, BlindFold R1CS instances become
structurally identical and fold directly.

## Two-tier aggregation

The proof stack spans two time horizons:

- **Setup-time.** $\pi_{\mathsf{SMTpolicy}}$ is produced once when the policy
  is authored. Only the proof (not the witness) is stored.
- **Per-action.** $\pi_{\mathsf{SMTplan}}$, $\pi_{\mathsf{SMTsolver}}$, and
  optionally $\pi_{\mathsf{NLplan}}$ are produced in real time.

Nova folding requires both witnesses simultaneously. If the policy proof was
generated a week ago and the witness was discarded, pure folding cannot
incorporate it.

### Tier 1: folding within a session

Per-action proofs are produced in the same session with all witnesses in
memory. They fold via Nova NIFS into a single `RelaxedR1CSInstance`, settled
by a single Spartan decider proof. This is the fast path.

### Tier 2: recursive verification across sessions

The setup-time proof $\pi_{\mathsf{SMTpolicy}}$ is verified *inside* the Tier
1 circuit: the per-action prover takes the serialised policy proof as a public
input and checks its validity, binding $\bar{P}_{\mathsf{SMT}}$ to the policy
input of $\pi_{\mathsf{SMTsolver}}$. Since the Jolt Atlas verifier is already
compressed to a single R1CS check + PCS opening via BlindFold, this recursive
subcircuit is compact.

### What the verifier sees

A single aggregated proof $\pi_{\mathsf{agg}}$ attesting:

1. A policy proof (setup-time, verified recursively) bound $\bar{P}$ to
   $P_{\mathsf{SMT}}$.
2. A translation proof (per-action, folded) bound the action to
   $C_{\mathsf{SMT}}$.
3. A solver proof (per-action, folded) checked
   $P_{\mathsf{SMT}} \wedge C_{\mathsf{SMT}}$ and returned verdict $b$.

The verifier checks one proof, sees the commitments and the verdict, done.

## Why not pure folding or pure recursion?

- Pure folding requires all witnesses simultaneously; doesn't work across time.
- Pure recursion adds overhead at every link, unnecessary for same-session
  proofs.
- Two-tier uses folding where cheap (same session) and recursion only where
  necessary (cross session). The setup-time proof is a reusable credential.

The runtime that enforces this split (Session, RecursiveCredential, Receipt)
is described on the [folding sessions](./folding-sessions.md) page.
