# Comparison with existing middleware

Every major agent framework now offers some form of tool-call interception:
a hook that fires between the LLM's decision to call a tool and the tool's
actual execution. This page surveys the landscape and identifies what
PreFlight provides that none of them do.

## The common pattern

All existing middleware follows the same flow:

```
LLM decides to call tool(args)
        │
        ▼
  ┌─────────────┐
  │  Middleware  │  ← inspect args, run checks
  │  (hook)     │
  └──────┬──────┘
         │
    allow / block / modify
         │
         ▼
  Tool executes (or not)
```

The check runs outside the LLM's reasoning loop, so the LLM cannot override
it via prompt injection. The check is deterministic (for rule-based systems)
or probabilistic (for LLM-based classifiers). The operator runs the
middleware and the consumer trusts the operator.

## Framework survey

### AWS Strands Agents

`BeforeToolCallEvent` hook fires before every tool call. The handler
inspects tool name and arguments and can allow, block, or modify. Rules are
deterministic and run outside the reasoning loop. Part of the Strands agent
framework; integrates with AgentCore.

### OpenAI Agents SDK

[Tool guardrails](https://openai.github.io/openai-agents-python/guardrails/)
wrap `function_tool` calls. Input guardrails run before execution with three
behaviors: `allow`, `reject_content` (block but continue with a message to
the model), or `raise_exception` (halt). Output guardrails run after
execution. Only covers `function_tool`; does not cover hosted tools (web
search, code interpreter, MCP tools).

### LangChain / LangGraph

Middleware hooks at three levels: before agent starts, after agent completes,
and around model/tool calls. Can require human approval for specific tools,
redact PII from inputs/outputs, or apply business rules. LangGraph adds node
guards that can block graph transitions.

### AEGIS

[AEGIS](https://arxiv.org/abs/2603.12621) is a framework-agnostic
pre-execution firewall. Three-stage pipeline:

1. **Deep string extraction** from tool arguments.
2. **Content-first risk scanning** (SQL injection, path traversal,
   data exfiltration patterns).
3. **Composable policy validation** against configurable rules.

Returns allow / block / pending. Pending routes to a human review dashboard
("Compliance Cockpit"). Tamper-evident audit trail with Ed25519 signatures
and SHA-256 hashes. Supports 14 agent frameworks in Python, JavaScript,
and Go.

Performance: 8.3 ms median interception latency, 1.2% false positive rate
on 500 benign tool calls, blocks all 48 attack instances in the evaluation
suite.

### Microsoft Agent Framework

Generic middleware pipeline for intercepting agent actions. Inject logging,
authentication, rate limiting, or custom logic at every step. Combined
Semantic Kernel + AutoGen. Less guardrail-specific than the others.

### AgentCore Gateway

Cedar policy engine + custom Lambda interceptors. The Gateway dispatches,
not the agent. See [AgentCore integration](./agentcore.md).

---

## What they all share

| Property | All existing middleware |
|----------|----------------------|
| Interception point | Between LLM decision and tool execution |
| Check type | Heuristic (regex, keyword, LLM classifier, rule engine) |
| Trust model | Trust the middleware operator |
| Proof of check | None, or signed logs (AEGIS) |
| Policy privacy | No (operator sees the policy; consumer may not) |
| Independent verifiability | No |
| Determinism | Rule-based: yes; LLM-based: no |

---

## How PreFlight differs

| Property | Existing middleware | PreFlight |
|----------|-------------------|-----------|
| **Check type** | Heuristic / rule-based / LLM classifier | Formal verification (SMT solver) |
| **Soundness** | Varies (AEGIS: blocks known attack patterns; LLM judges: 94-98%) | 99%+ on adversarial datasets |
| **Determinism** | Rule-based: yes; LLM-based: no | Yes (SMT solver is deterministic) |
| **Proof** | None, or Ed25519-signed log entries | Cryptographic zero-knowledge proof per check |
| **Independent verification** | No (must trust the operator) | Yes (anyone can verify the proof offline) |
| **Policy privacy** | No | Yes (policy can be a private witness) |
| **Prompt injection resistance** | Pattern-based (AEGIS scans for known attacks) | Structural (SMT operates on extracted claims, not raw text) |
| **What the proof attests** | "The middleware service claims it checked" | "This specific check ran on this specific input against this specific policy" |

---

## The key distinction

Every existing middleware answers the question: **"did some check run?"** The
operator says it did, and the consumer trusts the operator (or trusts a signed
log entry from the operator's service).

PreFlight answers a different question: **"here is a cryptographic proof that
this specific check ran on this specific input against this specific policy,
and the result was SAT or UNSAT."** The proof is independently verifiable by
anyone, without trusting the operator, without re-running the computation,
and without seeing the policy.

### AEGIS is the closest

AEGIS is the nearest competitor in spirit: framework-agnostic interception,
structured audit trail, Ed25519 signatures on log entries. But there is a
fundamental gap:

- AEGIS's "proof" is a **signed log entry**. It proves the AEGIS service
  *claims* it checked the action. It does not prove the check was correct,
  that the check ran the claimed policy, or that the input was not tampered
  with before checking. A compromised AEGIS service can sign a fake log
  entry.
- PreFlight's proof is a **zero-knowledge proof of SMT solver execution**.
  A compromised PreFlight service cannot forge a ZK proof. The proof is
  bound to the specific formula, the specific input, and the specific
  verdict. Forging requires breaking the underlying cryptographic
  assumption (discrete log on BN254), not compromising a signing key.

### Signed logs vs ZK proofs

| | Signed log (AEGIS) | ZK proof (PreFlight) |
|---|---|---|
| **What is trusted** | The signing key holder | The math (discrete log hardness) |
| **Key compromise** | Attacker can forge any log entry | No key to compromise; proofs are self-verifying |
| **What is bound** | "Service X claims it checked action Y" | "SMT formula Z was checked on input W with result b" |
| **Replay** | Possible (sign a log for a different action) | Impossible (proof is bound to specific formula + input) |
| **Policy binding** | No (log says "policy was checked," not which policy) | Yes (proof is relative to committed policy $\bar{P}$) |

---

## PreFlight is not a competing middleware; it is the proof layer

PreFlight does not replace any of these frameworks. Users stay on their
existing agent framework (LangChain, Strands, Microsoft Agent Framework,
OpenAI Agents SDK, or anything else). PreFlight is the **proof layer that
plugs into any of them** at the interception point they already provide.

The frameworks handle interception and dispatch. PreFlight handles
verification and attestation. The proof is what none of these frameworks
can produce on their own: they can block or allow, but they cannot *prove*
they blocked or allowed.

### The integration pattern

Every framework already provides a hook where custom logic runs before a
tool call executes. The integration is the same in all of them:

```
Framework hook fires (BeforeToolCallEvent / tool guardrail / function middleware)
    │
    ▼
Call PreFlight /v1/checkIt with the action text
    │
    ▼
Receive SAT/UNSAT + proof π
    │
    ├── SAT:  attach π to the action, let the framework proceed
    │
    └── UNSAT: block the tool call
```

| Framework | Hook | Where PreFlight call goes |
|-----------|------|--------------------------|
| Strands Agents (AWS) | `BeforeToolCallEvent` | Inside the event handler |
| OpenAI Agents SDK | Input tool guardrail | Inside the guardrail function |
| LangChain / LangGraph | Tool call middleware / node guard | Inside the middleware callback |
| Microsoft Agent Framework | Function middleware | Inside the `Invoke` wrapper |
| AEGIS | Policy validation stage | As an additional policy check |
| AgentCore Gateway | Request interceptor Lambda | Inside the Lambda function |

### What this gives the user

- **No framework migration.** The user keeps their existing agent stack.
- **Proof travels with the action.** The counterparty (or auditor) receives
  the action together with a cryptographic proof that a specific policy was
  checked on that specific input.
- **Framework-agnostic.** PreFlight is an HTTP API call. Any framework that
  supports pre-execution hooks can integrate it.
- **Layered defense.** The framework's own checks (heuristic rules, LLM
  classifiers, pattern scanners) run first (milliseconds). PreFlight runs
  second for high-stakes actions that need cryptographic proof (seconds).
  The two layers complement each other.

### What PreFlight adds that the hook alone cannot provide

The hook gives you a place to run a check. But the check itself is only as
trustworthy as the service that ran it. If the agent operator controls the
middleware, the operator can skip the check, fake the result, or swap the
policy.

PreFlight's ZK proof removes this trust assumption. The proof is bound to
the specific formula, the specific input, and the specific verdict. It is
independently verifiable by the counterparty without trusting the agent
operator, the middleware, or PreFlight's own infrastructure. The
counterparty verifies the math, not a signature.

The [two-layer policy design](./agentcore.md) in AgentCore reflects this
architecture: Cedar policies (Layer 1, fast, every call) for access control,
PreFlight proofs (Layer 2, proved, high-stakes calls) for cryptographic
guarantees.
