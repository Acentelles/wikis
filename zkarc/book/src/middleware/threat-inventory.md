# Threat inventory

A complete inventory of threats against the zkARc middleware, classified by
whether they are addressed, partially addressed, or not addressed by the
current design.

## Addressed

| Threat | How it's addressed |
|--------|-------------------|
| **Fabricated verdict.** Operator claims the policy was checked when it wasn't. | ZK proof. Cannot forge without breaking discrete log on BN254. |
| **Wrong policy.** Operator swaps in a permissive policy. | Policy commitment $\bar{P}$. The proof is bound to a specific committed policy. Counterparty checks $\bar{P}$ against an expected value. |
| **Tampered input.** Operator modifies the action text between extraction and solving. | Action text is a public input to the proof. The counterparty compares the proved action against what it received. |
| **Model swapping.** Operator substitutes a different extraction model. | Model commitment in $\pi_{\mathsf{SMTplan}}$. The proof binds the extraction to a specific committed model. |
| **Selective disclosure.** Operator reveals only favorable checks, hides failures. | Every action must carry a proof. No proof = counterparty rejects. |
| **Post-hoc plan fabrication.** Agent fabricates an action plan after the fact to get a favorable proof. | $\pi_{\mathsf{NLplan}}$ binds the plan to a prompt transcript and model execution. Requires forging a proof to fabricate. |
| **Re-running until SAT.** Operator repeatedly modifies claims until the solver returns SAT. | The proof binds the specific claims to the specific action text. Modified claims don't match the action the counterparty receives. |
| **Policy content leakage.** Counterparty learns the policy rules from the proof. | Hidden-policy mode: policy and certificate are private witnesses. Only the commitment and verdict are public. |

## Partially addressed

| Threat | What we do | What remains |
|--------|-----------|-------------|
| **Declared vs actual action.** Agent gets a proof for action X, executes action Y. | [RISC-V dispatch program](./riscv-dispatch.md): the dispatch instruction is a committed output of proved code. Host-side wrapper dispatches deterministically from that output. | The host could still not dispatch, or dispatch to a different endpoint. But the counterparty detects the mismatch by comparing the proved endpoint/action against what it received. |
| **Extraction correctness.** The SLM mistranslates the action, producing claims that don't reflect the action's semantics. | [Proven-SLM + proprietary agreement](../paper/translation-correctness.md): at least one model is proved to have run, and its verdict matches a proprietary model's verdict. | The proprietary model's verdict is taken on trust (private witness). A compromise of the provider's infrastructure could inject false agreement. |
| **Solver bugs.** The SMT solver has an implementation bug and returns a wrong verdict. | Multi-solver redundancy: oxiz (proved) + Z3/ARc (external, unproved) must agree. | Both solvers could share a common bug on the same formula. Unlikely for independent implementations but not impossible. |
| **Binary substitution.** Operator replaces the middleware binary with a modified version. | Binary commitment: the RISC-V binary is committed in the Jolt proof. A different binary produces a different commitment. | The counterparty must know what commitment to expect. Requires a reproducible build pipeline and a commitment registry. Without the registry, the counterparty can't distinguish a legitimate update from a malicious swap. |

## Not addressed

| Threat | Why not | What would close it |
|--------|---------|-------------------|
| **Compromised host machine.** Attacker with root access to the proving machine can intercept the guest output before the host wrapper dispatches, or modify the wrapper itself. | The ZKP protects computation inside the zkVM. Everything outside (host memory, network stack, OS) is unprotected. This is the [trust-your-own-setup](./middleware.md#trust-your-own-setup) floor. | TEE attestation on the host (hardware trust), or moving the entire middleware to a trusted third party's infrastructure (e.g., [AgentCore](./agentcore.md)). Neither eliminates trust; both shift it. |
| **Action fails after dispatch.** Network error, counterparty rejection, process crash mid-execution. | Availability problem, not integrity. The proof attests the dispatch instruction was produced; it cannot guarantee delivery. | Delivery receipts, idempotency tokens, on-chain settlement. Standard distributed-systems tooling. |
| **Policy quality.** The policy itself is wrong, incomplete, or malicious. The proof attests the policy was *enforced*, not that it was *good*. | Out of scope. "Is this a good policy?" is a reputation/audit question, not a cryptographic one. | [Hidden-policy trust spectrum](../paper/hidden-policy-trust.md): auditor-mediated trust, meta-property proofs, or reputation-backed commitment. |
| **Prompt injection on the planner.** An adversary injects a malicious prompt that causes the planner LLM to produce a harmful action plan that happens to be policy-compliant. | The proof attests the policy was checked. If the action is SAT under the policy, it passes, regardless of how the action was produced. The policy is the defense, not the proof. | Better policies (more rules, tighter constraints). Or $\pi_{\mathsf{NLplan}}$ to bind the plan to a specific prompt transcript, letting the principal audit what prompt produced the plan. |
| **Side-channel leakage from proofs.** The proof itself, even in hidden-policy mode, could leak information about the policy structure (e.g., proof size correlates with policy complexity, proving time correlates with rule count). | Not currently mitigated. Proof size and proving time are observable. | Padding proofs to a fixed size. Adding random delay to proving time. Both add cost. |
| **SRS compromise.** The HyperKZG trusted setup ceremony produces a structured reference string with toxic waste. If the toxic waste is not destroyed, an attacker can forge proofs. | Standard trusted-setup assumption. Same as any KZG-based system. | Use a transparent PCS (Dory, FRI) instead of HyperKZG. Or a multi-party ceremony where forgery requires all participants to collude. |
| **Quantum adversary.** The discrete-log assumption on BN254 is broken by a quantum computer. All proofs become forgeable. | Not addressed. Same as every elliptic-curve-based ZK system deployed today. | Post-quantum ZK proving systems (lattice-based SNARKs). Active research area, not yet practical for this use case. |
