# AgentCore integration

[Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) is an
agentic platform for building, deploying, and operating AI agents at scale.
Its architecture maps almost exactly onto the zkARc middleware design: a
Gateway that intercepts every outbound tool call, Lambda-based interceptors
for custom logic, a built-in policy engine, and dispatch control that lives
in the Gateway rather than in the agent.

## Why AgentCore is the natural host

### The Gateway is the middleware

AgentCore Gateway sits between the agent and external tools/APIs. Every tool
call passes through it. The Gateway already provides:

- **Interception.** Request interceptor Lambdas process each outbound call
  before it reaches the target tool. Response interceptor Lambdas process
  the response before it returns to the agent.
- **Dispatch control.** The Gateway executes the tool call, not the agent.
  This solves the
  ["gating is not enough"](./middleware.md#gating-is-not-enough-the-middleware-must-execute)
  problem: the Gateway is the component that dispatches, so if the
  interceptor blocks, the action never fires.
- **Identity and credentials.** The Gateway handles authentication to
  external tools. The agent never sees raw credentials.
- **Audit logging.** Built-in observability. Proofs can be logged alongside
  tool call traces.

### Cedar + zkARc: two-layer policy

AgentCore already has a
[policy engine](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy.html)
that evaluates
[Cedar](https://www.cedarpolicy.com/) policies against each tool call. Cedar
is fast, deterministic, and handles access control, rate limiting, and basic
allow/deny rules. Policies can be written in natural language and compiled
to Cedar automatically.

zkARc adds a second layer on top:

| | Cedar (Layer 1) | zkARc (Layer 2) |
|---|---|---|
| **Speed** | Milliseconds | Seconds |
| **Formalism** | Cedar (access control) | SMT-LIB (formal logic) |
| **Guarantees** | Deterministic evaluation | Cryptographic proof |
| **Privacy** | Policy visible to operator | Policy can be hidden |
| **Trust model** | Trust the Gateway operator | Trustless (ZKP) |
| **When to use** | Every tool call | High-stakes actions |

Layer 1 (Cedar) is always on. Layer 2 (zkARc) is opt-in for actions where
the customer needs cryptographic guarantees: financial transactions,
regulated actions, cross-organisational agent commerce.

## Integration architecture

The zkARc request interceptor Lambda is the hook:

```
Agent (on AgentCore Runtime)
    │
    │ tool call
    ▼
AgentCore Gateway
    │
    ├── Cedar Policy Engine (existing, fast, no ZK)
    │
    ├── zkARc Request Interceptor (Lambda)
    │       │
    │       ├── Extract claims (inline classifier or SageMaker call)
    │       │
    │       ├── Call Jolt proving service (ECS/Fargate)
    │       │       │
    │       │       ├── oxiz in Jolt: Solve(P_SMT ∧ C_SMT) = b
    │       │       │
    │       │       └── Return (b, π)
    │       │
    │       ├── If b = SAT: attach π to request metadata, allow
    │       │
    │       └── If b = UNSAT: deny request, return error to agent
    │
    ▼
External Tool / Counterparty API
    │
    └── (proof π available for offline verification)
```

### Step by step

1. The agent issues a tool call (action $A$ with parameters and context).
2. The Gateway routes it through the Cedar policy engine first. If Cedar
   denies, the call is blocked immediately (fast path).
3. If Cedar allows, the zkARc request interceptor Lambda fires.
4. The Lambda extracts claims $C_{\mathsf{SMT}}$ from the action. For a tiny
   classifier, this can run inline in the Lambda. For a larger SLM, it calls
   a SageMaker endpoint.
5. The Lambda calls a Jolt proving service (ECS/Fargate task) with the
   formula $P_{\mathsf{SMT}} \wedge C_{\mathsf{SMT}}$. The proving service
   runs oxiz inside Jolt and returns the verdict $b$ and proof $\pi$.
6. If $b = \mathsf{SAT}$ and the proof is valid, the Lambda attaches $\pi$
   to the request metadata and returns the request to the Gateway for
   dispatch to the external tool.
7. If $b = \mathsf{UNSAT}$ or proof generation fails, the Lambda returns a
   deny response. The tool call never fires.
8. The counterparty (or an auditor) can later verify $\pi$ independently
   using the verifier SDK.

## What AgentCore provides that we don't build

| Capability | AgentCore provides | We build |
|-----------|-------------------|----------|
| Interception point | Gateway interceptor Lambdas | The Lambda function itself |
| Dispatch control | Gateway executes the tool call | Nothing (already solved) |
| Identity / credentials | Gateway credential management | Nothing |
| Audit trail | Built-in observability | Proof logging (attach $\pi$ to traces) |
| NL policy authoring | NL-to-Cedar compilation | NL-to-SMT-LIB compilation (ARc PMC) |
| Fast access control | Cedar policy engine | Nothing (Layer 1 stays) |

## What we build

1. **zkARc interceptor Lambda.** Receives the tool call, extracts claims,
   calls the proving service, gates on the result. Lightweight: the Lambda
   itself is just an orchestrator.

2. **Jolt proving service.** An ECS/Fargate task (or dedicated EC2) that runs
   oxiz inside Jolt. The Lambda calls it via HTTP. The proving service is
   stateless: it takes a formula and returns a verdict + proof.
   - Lambda's 15-min timeout and 10GB memory limit may be tight for large
     proofs, but the ~2s solver proof for 50 rules should fit comfortably
     in a Lambda-to-Fargate call pattern.

3. **Claim extraction model.** A tiny classifier or SLM deployed as a
   SageMaker endpoint (or inline in the Lambda for classifiers small enough).
   Produces $C_{\mathsf{SMT}}$ from the action text.

4. **Policy compilation tooling.** A CLI or web UI that takes a
   natural-language policy document and produces $P_{\mathsf{SMT}}$ +
   commitment $\bar{P}$. This is ARc's PMC, adapted for zkARc. Runs once
   per policy update, offline.

5. **Verifier SDK.** A Rust library (with FFI bindings for Python/Node) that
   counterparties or auditors use to check proofs offline. Could also be
   deployed as a Lambda function the counterparty calls.

## Why this is a good pitch

- **No new infrastructure paradigm.** zkARc plugs into an existing
  interception point that AWS already built and maintains. Every agent on
  AgentCore already routes through the Gateway.
- **Complements, doesn't replace.** Cedar policies stay for fast access
  control. zkARc adds the cryptographic layer for high-stakes actions.
- **Their customers are asking.** Banks, stablecoin providers, and regulated
  enterprises using AgentCore want audit proofs. zkARc turns the audit trail
  from "we logged what happened" to "here's a cryptographic proof of what
  happened."
- **ARc integrates directly.** ARc's policy compilation (PMC) produces the
  SMT-LIB policies. AgentCore's natural-language policy feature could use
  ARc under the hood to compile NL policies into SMT-LIB, then prove them
  via zkARc. The ARc team is already a collaboration partner.
- **Revenue model.** Proof generation is a compute-intensive service.
  Customers pay per proof (or per agent instance with proofs enabled).
  The proving service runs on AWS infrastructure, generating AWS compute
  revenue alongside the zkARc service fee.

## Open questions

1. **Lambda vs sidecar.** Should the interceptor be a Lambda (serverless,
   pay-per-invocation) or a sidecar container running alongside the agent
   in ECS (always-on, lower latency for frequent calls)?

2. **Proof attachment format.** How does $\pi$ travel with the tool call?
   Options: Gateway metadata field, custom HTTP header, or a separate
   proof-store service that the counterparty queries by action ID.

3. **SRS management.** The HyperKZG structured reference string must be
   available to both the proving service and the verifier. Host it as an
   S3 artifact? Bundle it with the proving service container? Distribute
   via a well-known URL?

4. **Multi-tenant proving.** If the proving service serves multiple
   customers, each with different policies, the service must handle policy
   isolation. Each customer's $\bar{P}$ and $P_{\mathsf{SMT}}$ are private
   witnesses; the proving service must not leak one customer's policy to
   another.

5. **Latency budget.** The ~2s proving overhead is acceptable for
   high-value financial transactions but may be too slow for
   high-frequency, low-value actions. The two-layer design (Cedar fast
   path + zkARc slow path) mitigates this: only actions flagged by Cedar
   as requiring ZK proof go through the proving service.
