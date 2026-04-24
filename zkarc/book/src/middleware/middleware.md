# Middleware

The middleware is the enforcement layer that closes the gap between "the
guardrail was checked" and "the checked action is the one that executed."

## Core idea

A middleware component intercepts actions from an agent, requiring that each
action is accompanied by a zero-knowledge proof. The middleware verifies the
proof and only dispatches the action if the proof is valid. The counterparty's
system rejects any action that arrives without a valid proof.

The middleware itself does not need to be trusted by the counterparty, because
the proof is the trust anchor.

## Gating is not enough: the middleware must execute

A critical design constraint: the middleware cannot merely *gate* the action
(verify the proof and then tell the agent "go ahead"). If the middleware only
gates, the agent retains control of dispatch and can still execute a different
action after receiving approval. The proof attests that a guardrail was
checked, but nothing binds the checked action to the action that actually
fires.

The middleware must be the component that *executes* the action. The proof
chain is then:

- **Proof A:** a guardrail ran on this input (the policy check).
- **Proof B:** this middleware code ran (including the dispatch call).

If Proof B triggers the action, the binding is cryptographic: the action is a
deterministic consequence of the proved computation. No separate "the action
actually happened" attestation is needed, because the dispatch *is* a side
effect of the proved execution.

If the middleware is small enough, it can itself run inside a zkVM (Jolt). Then
Proof B literally attests: "this deterministic program, which includes the
dispatch call, was executed on these inputs." The counterparty verifies Proof B
and knows that dispatch happened as a consequence of proved, policy-checked
code.

The remaining gap after this design is *availability*, not *integrity*: the
action could fail midway (network error, counterparty rejection, process
crash). But that is a distributed-systems problem, not a cryptographic one.

## Two cases

The middleware design differs depending on what is being proved:

### Case 1: [zk(SMT) middleware](./zk-smt.md)

The simplest case. The agent runs the SMT solver inside Jolt, produces a proof,
and sends the proof together with the action to the middleware. The middleware
verifies and dispatches.

**Feasibility:** buildable now with the existing Jolt codebase.

### Case 2: [zkML middleware](./zkml-wrapping.md)

The harder case. The middleware wraps the LLM so that every input to the agent
is an input to the middleware. The middleware runs the model, proves the
execution, runs the extraction + solver pipeline, and dispatches only if the
policy passes.

For the zkML case, classification models are a natural fit: they are tiny,
work with Jolt Atlas today, and the middleware only needs to classify the
action (compliant / non-compliant) rather than generate text. This avoids
the cost of proving a full autoregressive LLM.

**Feasibility:** feasible for small/classification models (sub-second). Heavy
for large generative models (14-38s for GPT-2). Pragmatic v1: wrap only the
extraction step (small SLM or classifier) + solver.

## Deployment models

Three deployment tiers, each suited to a different customer segment:

### Managed service (e.g., within an agent orchestration platform)

The middleware lives inside an existing agent orchestration platform as a
service component. Customers spin up agent instances with guardrail proofs
enabled. The platform handles Jolt compilation, SRS management, and proof
generation. Customers (financial institutions, regulated enterprises) get
proofs for auditors as a platform feature.

This is the enterprise path. The proving infrastructure is heavy (Jolt
compilation, RISC-V target, large SRS); it naturally centralises toward
whoever already runs agent infrastructure. If an orchestration platform
already offers guardrails as a service, adding proof generation is a natural
extension.

### Self-hosted (local Jolt)

The customer runs Jolt locally on their own infrastructure. More setup
burden, but gives the customer full control over the proving environment and
avoids trusting a third-party prover with policy content or action data.

Suited to crypto-native organisations or customers who already operate
specialised infrastructure. Less suited to customers who want a managed
experience.

### Open-source hook

Ship the middleware as a free, open-source component that any AI provider can
integrate. The middleware is provider-agnostic: the user chooses which AI
provider they want, and the middleware wraps it. This is the adoption and
ecosystem play, not a direct revenue play, but it establishes the proof format
as a standard.

The key constraint across all three: the proving infrastructure (Jolt
compiler, zkVM runtime, SRS) is heavy. Some systems cannot handle it. This
pushes deployment toward managed services or dedicated infrastructure rather
than lightweight client-side integration.

## Trust-your-own-setup

No proof chain can protect against an attacker with physical or root access
to the machine running the middleware. The ZKP guarantees that *if the
middleware ran as compiled*, then the action was policy-checked and dispatched
correctly. It does not guarantee the middleware binary was not tampered with
before execution.

The trust model has a hard floor: whoever operates the middleware must be
trusted not to have modified the binary. The proof attests to faithful
execution of a *committed program*, but the commitment is to a specific
compiled binary. If the operator swaps the binary, the commitment changes,
and the proof will not verify against the expected commitment, but only if
the verifier knows what commitment to expect.

This means:

- **The middleware binary must be committed and verifiable.** A reproducible
  build pipeline ensures the commitment is deterministic and auditable. The
  verifier (or a registry) holds the expected binary commitment. Any change
  to the middleware code produces a different commitment, which the verifier
  detects.

- **Trust in the operator is bounded.** The operator is trusted to run the
  committed binary on committed hardware. The ZKP removes trust in the
  *execution* (correct inputs, correct policy, correct dispatch). What
  remains is trust in the *deployment* (the right binary is running). This
  is analogous to TEE-based approaches, where you trust the hardware
  manufacturer; here you trust the build pipeline.

- **You trust your own setup at a minimum.** This is the irreducible trust
  assumption. An agent operator who deploys the middleware on their own
  infrastructure trusts that their own machine is not compromised. An
  enterprise customer who uses a managed service trusts the service
  operator's deployment. In both cases, the ZKP upgrades the trust story
  from "trust everything the operator claims" to "trust only that the
  operator ran the committed binary."

For the paper, the threat model section should state this clearly: zkARc
assumes the middleware binary is committed and verifiable via a reproducible
build hash, and that an attacker with root access to the proving machine can
subvert the system by replacing the binary. The proof does not protect against
binary substitution; it protects against everything else.
