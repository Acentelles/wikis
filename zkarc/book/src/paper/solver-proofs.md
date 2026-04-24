# Solver proofs

## Two instantiation options for $\pi_{\mathsf{SMTsolver}}$

### Full solver in Jolt

Oaksive (a Rust SMT solver) is compiled to RISC-V and runs inside Jolt. The
proof attests to the entire execution trace, including the solver's bytecode
commitment (solver-execution binding).

**Proving budget.** For policies up to 50 rules (matching real-world focused
policy documents), Oaksive completes within the RISC-V cycle budget that Jolt
can prove in approximately 2 seconds. The rule count is the binding constraint
on policy complexity.

**Multi-solver redundancy.** Running the solver inside a zkVM guarantees
faithful execution but not correctness (implementation bugs). To address this,
an external trusted solver (Z3 or the ARc pipeline) independently checks the
same formula. The ZK proof attests that the proven solver's verdict matches the
external verdict (supplied as a private witness). Same proven-anchor +
agreement pattern as [translation correctness](./translation-correctness.md).

### Checker-based instantiation

The solver runs natively (outside ZK) and emits a certificate. A small checker
verifies the certificate against the formula, and only the checker runs inside
the zkVM. The certificate is a private witness (important for policy hiding).

**Candidate formats:**
- **LRAT** for propositional UNSAT: well-defined, near-linear checkers, several
  compact C/Rust implementations suitable for RISC-V compilation.
- **Alethe** for SMT: emerging standard, less mature than LRAT. Coverage of
  QF_NRIA (the theory used by ARc) is an open question.
- **Pragmatic intermediate:** use a SAT-level certificate after theory
  reasoning reduces the problem to a propositional core.

### When to use each

| Criterion | Full solver | Checker-based |
|-----------|------------|---------------|
| Policy size | Small (within proving budget) | Large (expensive to solve) |
| Binding requirement | Solver-execution binding needed | Verdict binding sufficient |
| Certificate format | Not required | Stable format available |

In either case, the ZK proof *is* the portable, succinct, standardised proof
format. It sidesteps the proof-format standardisation debate in the SMT
community entirely. zk(SMT) is the proof format.
