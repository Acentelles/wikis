# Threat model

## Setting

Agentic commerce: autonomous agents negotiate, transact, and execute on behalf
of principals. A seller needs to know the funds are valid and the buyer's agent
was authorised. A buyer needs to know their agent didn't go off the rails and
the seller's system enforced the claimed policies.

## Untrusted prover

We assume an untrusted prover operates the agent and guardrail infrastructure.
The prover may attempt to:

1. Fabricate that a policy was checked when it was not.
2. Reveal only a favourable subset of the computation.
3. Hide the provenance of the agent's action.

The verifier should validate a succinct proof without re-running any model
inference or SMT pipeline.

## Execution binding: declared vs actual actions

The fundamental gap: if an agent declares one action, gets a valid proof, and
then executes a different action, the proof is useless. This is not specific
to zkARc; it applies to any declaration-based system.

**Partial mitigation via $\pi_{\mathsf{NLplan}}$.** If the planner model's
output is proved (bound to a prompt transcript and model execution), fabricating
a post-hoc plan requires forging a proof. This raises the bar from "the agent
lied about what it planned" to "the agent faithfully planned X and then chose
not to do X."

**Full closure via middleware.** An enforcement-layer middleware intercepts
every outbound action and requires a valid zkARc proof before dispatch. The
counterparty's system rejects unproven actions. The binding is between the
*proven plan* and the *action as received by the counterparty*. The remaining
gap (received vs executed downstream) is a distributed-systems problem, not a
cryptographic one. See the [middleware section](../middleware/middleware.md).
