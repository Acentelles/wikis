# zkARc

zkARc extends [Automated Reasoning Checks (ARc)](https://arxiv.org/abs/2511.09008),
a neurosymbolic system that translates natural-language policies into formal
logic and verifies AI-generated content via SMT solvers, with succinctly
verifiable zero-knowledge proofs.

The paper source is at `my/papers/zkARc/main.tex` and syncs with Overleaf.

## Why zkARc

In agentic commerce, autonomous agents negotiate, transact, and execute on
behalf of principals. Money, data, and security are on the line. Each party
needs assurance that the counterparty's guardrails actually ran, that they ran
on the declared input, and that the verdict is authentic, all without trusting
the counterparty's infrastructure or seeing its policies.

Existing guardrail frameworks (LangChain, NeMo Guardrails, Llama Guard) are
centralized and heuristic. They provide no cryptographic binding between a
guardrail check and the action it governs.

zkARc provides two properties that no prior guardrail system offers
simultaneously:

- **Binding.** An unforgeable proof that the guardrail pipeline executed
  correctly on the declared input.
- **Hiding.** The ability to keep policies, action text, and even the model
  itself private from the verifier.

## Contributions

1. A modular proof-carrying guardrail architecture where each pipeline step
   produces an independent ZK proof, and the security claim is parameterised
   by which subset of proofs is instantiated.
2. A two-tier proof aggregation design: Nova folding for same-session proofs,
   recursive verification for cross-session (setup-time) proofs, producing a
   single succinct receipt.
3. Model and policy hiding at negligible overhead for inputs and weights, with
   architecture hiding via recursive verification.
4. A proven-SLM translation correctness pattern that proves a small model ran
   in ZK and checks agreement with a proprietary model.
5. An enforcement-layer middleware that captures the agent's declared action,
   verifies the policy proof, and gates dispatch on proof validity.

## Collaboration

Joint work with the AWS ARC team (Ben, Aman). AWS owns the QF_NRIA formalism
and Section 3 (Preliminaries) of the paper. We own the ZK architecture
(Sections 4-6) and the middleware design.

## How this wiki is organised

- **The paper** distils each section of the LaTeX draft into a standalone page.
- **Middleware** documents the enforcement-layer architecture, starting from
  the simplest case (zk(SMT) gate) through the full zkML wrapping.
