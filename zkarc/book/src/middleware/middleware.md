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

## Two cases

The middleware design differs depending on what is being proved:

### Case 1: [zk(SMT) middleware](./zk-smt.md)

The simplest case. The agent runs the SMT solver inside Jolt, produces a proof,
and sends the proof together with the action to the middleware. The middleware
runs the Jolt verifier. If the proof checks out, the action is dispatched.

The agent does the proving (expensive). The middleware does the verifying
(cheap, sub-second). The middleware is stateless and thin.

**Feasibility:** buildable now with the existing Jolt codebase.

### Case 2: [zkML middleware](./zkml-wrapping.md)

The harder case. The middleware wraps the LLM so that every input to the agent
is an input to the middleware. The middleware runs the model, proves the
execution, runs the extraction + solver pipeline, and dispatches only if the
policy passes.

This solves the self-awareness problem (the agent can't prove its own inference
after the fact) but changes the deployment model: the middleware becomes the
execution environment, not just a gate.

**Feasibility:** feasible for small models (1-2s SLM). Heavy for large models
(14-38s for GPT-2). Pragmatic v1: wrap only the extraction step + solver.

## Remaining gap

After the middleware, the binding is between the *proven plan* and the
*action as received by the counterparty*. The gap between "received" and
"actually executed downstream" is a distributed-systems problem (idempotency,
delivery guarantees), not a cryptographic one.
