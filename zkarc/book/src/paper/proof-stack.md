# Proof-carrying guardrails

zkARc should be understood not as a single proof but as a modular stack.
The number and placement of proofs determine exactly which claims a third
party can verify without trusting the agent operator.

## The proof stack

$$
\Pi = \{\pi_{\mathsf{NLplan}},\; \pi_{\mathsf{SMTpolicy}},\; \pi_{\mathsf{SMTplan}},\; \pi_{\mathsf{SMTsolver}}\}
$$

| Proof | What it proves | System |
|-------|---------------|--------|
| $\pi_{\mathsf{NLplan}}$ | Planner model produced action plan $A_{\mathsf{NL}}$ from prompt $Q$ | Jolt Atlas |
| $\pi_{\mathsf{SMTpolicy}}$ | Policy compiler produced $P_{\mathsf{SMT}}$ from $P_{\mathsf{NL}}$ | Jolt Atlas |
| $\pi_{\mathsf{SMTplan}}$ | Answer verifier extracted $C_{\mathsf{SMT}}$ from $A_{\mathsf{NL}}$ | Jolt Atlas |
| $\pi_{\mathsf{SMTsolver}}$ | Solver checked $P_{\mathsf{SMT}} \wedge C_{\mathsf{SMT}}$ and returned $b$ | Jolt |

## Proof stack subsets

The security claim is parameterised by which proofs are instantiated:

- **Only $\pi_{\mathsf{SMTsolver}}$.** Trust removed from solver execution
  only.
- **$\pi_{\mathsf{SMTplan}} + \pi_{\mathsf{SMTsolver}}$.** Additionally
  certifies the logical claims correspond to the declared action plan.
- **$\pi_{\mathsf{SMTpolicy}} + \pi_{\mathsf{SMTplan}} + \pi_{\mathsf{SMTsolver}}$.**
  The core end-to-end claim: policy was formalised, action was extracted,
  solver checked correctly.
- **Adding $\pi_{\mathsf{NLplan}}$.** Binds the action plan to a specific
  prompt and model execution.

## Setup-time vs per-action proofs

Policy compilation (PMC) is typically done offline and is internal to the
policy owner. $\pi_{\mathsf{SMTpolicy}}$ can be computed once and cached. The
ZK overhead that determines real-world latency is concentrated in the per-action
proofs $\pi_{\mathsf{SMTplan}}$ and $\pi_{\mathsf{SMTsolver}}$.

This asymmetry drives the [two-tier aggregation](./proof-aggregation.md)
design.

## The minimal end-to-end claim

For agentic commerce, the minimal set is:

$$
\pi_{\mathsf{SMTpolicy}} + \pi_{\mathsf{SMTplan}} + \pi_{\mathsf{SMTsolver}}
$$

This yields: "the NL policy was compiled into the formal policy used for
checking; the NL action was compiled into the logical claims used for checking;
and the solver correctly determined the compliance verdict."
