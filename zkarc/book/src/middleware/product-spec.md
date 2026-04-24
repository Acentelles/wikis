# Product spec

What the middleware looks like as a shippable product.

## What the product is

A service (or self-hosted binary) that sits between an AI agent and the
outside world. Every outbound action from the agent passes through it. The
middleware proves the action was policy-checked, and only then dispatches it.
The counterparty receives the action together with a proof.

---

## The software stack

### 1. Policy compiler (setup-time, runs once)

| | |
|---|---|
| **Input** | Natural-language policy document (e.g., "never authorize payments above \$10K without manager approval") |
| **Process** | ARc's PMC translates the document to SMT-LIB. Optionally, Jolt Atlas proves the translation ($\pi_{\mathsf{SMTpolicy}}$). |
| **Output** | Committed formal policy $\bar{P}$ + cached policy proof |
| **Software** | ARc's translation model (or a fine-tuned SLM) + Jolt Atlas prover |
| **Runs** | Once per policy update. Offline. |

### 2. Middleware runtime (per-action, hot path)

The core product. Five components:

**Interceptor.** Hooks into the agent's outbound call path. Could be:
- An HTTP proxy (the agent sends requests to `localhost:PORT` instead of the
  counterparty).
- An SDK wrapper (a function call that wraps the agent's dispatch).
- A plugin for agent frameworks (LangChain, CrewAI, AutoGen).

The agent sends its intended action to the middleware instead of directly to
the counterparty.

**Claim extractor.** Takes the action text and extracts SMT claims. Options:
- A small classification model (sub-second, provable in Jolt Atlas). Produces
  a binary verdict (compliant / non-compliant) rather than a full translation.
- A small generative SLM that produces a full SMT translation (1-2s, also
  provable in Jolt Atlas).
- Hardcoded extraction rules (for v0; not provable, extraction is trusted).

**SMT solver.** oxiz compiled to RISC-V, running inside Jolt. Checks the
extracted claims against the committed policy. Produces
$\pi_{\mathsf{SMTsolver}}$.

**Proof aggregator.** Folds the extraction proof and solver proof (and
optionally the cached policy proof via recursive verification) into a single
$\pi_{\mathsf{agg}}$. See
[two-tier aggregation](../paper/proof-aggregation.md).

**Dispatcher.** If the aggregated proof verifies and the verdict is SAT,
dispatches the action to the counterparty together with the proof. If not,
blocks and returns an error to the agent. The dispatcher is the enforcement
point: the action fires as a consequence of proved, policy-checked code.

### 3. Verifier (counterparty-side)

| | |
|---|---|
| **Input** | Action + proof + policy commitment |
| **Process** | Runs the Jolt / Jolt Atlas verifier (~500ms) |
| **Output** | Accept or reject |
| **Software** | Lightweight verifier binary, SDK, or on-chain contract. Stateless. |

---

## What you'd ship

| Component | Form factor | Who runs it |
|-----------|------------|-------------|
| Policy compiler | CLI tool or web UI | Policy owner (once per policy update) |
| Middleware runtime | Docker container, cloud service, or agent-framework plugin | Agent operator |
| Verifier | Rust library, npm package, or Solidity contract | Counterparty |

---

## Dependencies

| Dependency | Role | Notes |
|-----------|------|-------|
| **Jolt** | RISC-V zkVM for solver proof | Needs RISC-V compilation target |
| **Jolt Atlas** | ONNX zkML for extraction/classification proof | Needs ONNX model export |
| **oxiz** | Rust SMT solver | Compiled to RISC-V for in-circuit solving |
| **ARc or equivalent SLM** | Claim extraction | ARc's proprietary model (agreement-check) or fine-tuned open model |
| **HyperKZG SRS** | Polynomial commitment scheme | Generated once, shared across all proofs |

---

## Implementation phases

### v0: zk(SMT) proxy (minimum viable product)

The smallest shippable thing.

1. A Docker container that runs as an HTTP proxy.
2. The agent sends POST requests to the middleware instead of the counterparty.
3. The middleware extracts claims from the action using hardcoded extraction
   rules (no ML, extraction is trusted).
4. The middleware runs oxiz inside Jolt, produces
   $\pi_{\mathsf{SMTsolver}}$.
5. The middleware forwards the request to the counterparty with the proof
   attached as an HTTP header (or JSON envelope).
6. The counterparty has a verifier endpoint that checks the proof before
   accepting the action. (Or ignores the proof for now, during integration.)

**What v0 proves:** the solver ran on these claims against this policy.

**What v0 does not prove:** that the claims were correctly extracted from the
action (extraction is trusted), or that the action plan came from a specific
model.

**Proving overhead:** ~2s per action (solver only).

**What to build:**
- Jolt verifier library callable from the middleware (Rust, exposed via FFI
  or HTTP).
- Serialisation format for $(A, b, \pi, \bar{P})$.
- HTTP proxy with dispatch gate.
- Policy commitment management (store $\bar{P}$, check against a registry).

### v1: proved extraction

Add a small classification model or SLM for claim extraction, proved via
Jolt Atlas. The middleware now produces $\pi_{\mathsf{SMTplan}}$ (extraction)
+ $\pi_{\mathsf{SMTsolver}}$ (solver), folded into $\pi_{\mathsf{agg}}$.

**What v1 adds:** cryptographic binding between the action text and the SMT
claims. No one can fabricate favourable claims without the proof failing.

**Proving overhead:** ~3-4s per action (extraction + solver).

### v2: policy proof integration

Add the cached policy proof ($\pi_{\mathsf{SMTpolicy}}$) via Tier 2
recursive verification. The aggregated proof now covers the full pipeline
from natural-language policy to solver verdict.

**What v2 adds:** the verifier knows the formal policy came from the declared
natural-language policy, not a substituted one.

### v3: full LLM wrapping

The middleware wraps the planner LLM and proves $\pi_{\mathsf{NLplan}}$. The
agent's action plan is bound to a specific prompt and model execution.

**What v3 adds:** the verifier knows the action plan was not fabricated
post-hoc.

**When v3 makes sense:** when the planner is a small model (sub-10s proving)
or when hardware acceleration makes large-model proving practical.

---

## Architecture diagram

```
                    ┌─────────────────────────────────────────┐
                    │           MIDDLEWARE RUNTIME             │
                    │                                         │
Agent ──(action)──► │  Interceptor                            │
                    │      │                                  │
                    │      ▼                                  │
                    │  Claim Extractor ──── π_SMTplan (v1+)   │
                    │      │                                  │
                    │      ▼                                  │
                    │  oxiz (in Jolt) ── π_SMTsolver       │
                    │      │                                  │
                    │      ▼                                  │
                    │  Proof Aggregator                       │
                    │      │  ┌──────────────────────┐        │
                    │      │  │ π_SMTpolicy (cached)  │ (v2+) │
                    │      │  │ (recursive verify)    │        │
                    │      │  └──────────────────────┘        │
                    │      ▼                                  │
                    │  Dispatcher ──── if SAT + verified ─────┼──► Counterparty
                    │      │                                  │    (action + π_agg)
                    │      └──── if UNSAT or failed ──────────┼──► Agent (error)
                    │                                         │
                    └─────────────────────────────────────────┘
                                                               
                                        Counterparty
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │   Verifier    │
                                    │  (~500ms)     │
                                    │               │
                                    │  accept/reject│
                                    └───────────────┘
```

---

## Open engineering questions

1. **Proof serialisation format.** What wire format for $(A, b, \pi, \bar{P})$?
   JSON envelope with base64-encoded proof? Protobuf? A custom binary format?
   Needs to be compact (proofs are ~KB range) and parseable by the verifier SDK.

2. **Agent framework integration.** For LangChain/CrewAI/AutoGen: is the
   interceptor a tool wrapper, a callback, or a transport-level proxy? Each
   framework has different hook points.

3. **SRS distribution.** The HyperKZG SRS must be available to both prover
   (middleware) and verifier (counterparty). Ship it as a downloadable
   artifact? Embed it in the Docker image? Host it at a well-known URL?

4. **Policy commitment registry.** The counterparty needs to know what
   $\bar{P}$ to expect. Is there a shared registry? Does the policy owner
   publish $\bar{P}$ out-of-band? Or does each action carry $\bar{P}$ and the
   counterparty checks it against an on-chain or off-chain anchor?

5. **Reproducible builds.** The trust-your-own-setup principle requires a
   deterministic build pipeline. The Jolt-compiled RISC-V binary must be
   reproducible so the binary commitment is auditable. Nix, Docker with
   pinned layers, or a custom build system?

6. **Multi-policy support.** An agent may need to satisfy multiple policies
   (internal compliance + counterparty requirements). Each produces a separate
   proof. The middleware must check all of them and the aggregator must fold
   all of them.
