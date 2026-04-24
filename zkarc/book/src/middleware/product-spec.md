# Product spec

## PreFlight today

[ICME PreFlight](https://docs.icme.io/) is the deployed production system
built on the zkARc research. It is a pay-per-use API that agents call to
check actions against policies:

| Endpoint | Cost | What it does |
|----------|------|-------------|
| Policy compilation | $3.00 (one-time) | NL policy → SMT-LIB + consistency check |
| `/v1/checkRelevance` | Free | Screens action against policy variables; determines if paid check is needed |
| `/v1/checkIt` | $0.01 | Full SAT/UNSAT evaluation with cryptographic proof |
| `/v1/verifyProof` | Free (single-use) | Public verification of a proof receipt |

### What PreFlight already provides

- **Policy compilation.** Natural-language rules compiled to SMT-LIB formal
  logic. Consistency checking catches contradictions before deployment.
- **Dual verification.** Local SMT solver + cloud ARc engine independently
  check the same action. Both must return SAT for clearance.
- **ZK proofs via Jolt.** Every `checkIt` call produces a cryptographic proof
  receipt, downloadable and independently verifiable.
- **x402 crypto micropayments.** Autonomous agents can pay per check with
  USDC on Base, enabling fully autonomous agent-to-agent commerce.
- **Three integration patterns.** Tool call interception, skill-directed
  description, and planning step interception.
- **99%+ soundness** on adversarial datasets, deterministic (same input
  produces same result), resistant to prompt injection on verifiable facts.

### The gap: gating vs executing

PreFlight today is a **check API**: the agent calls `/v1/checkIt`, receives a
verdict, and then *decides* whether to proceed. The agent retains control of
dispatch. A compliant agent will respect an UNSAT verdict, but a compromised
or adversarial agent can ignore it.

The proof binds the verdict to the action, but nothing binds the action to
what actually executes. The agent could call `checkIt`, get SAT, and then
execute a different action.

---

## PreFlight v2: the middleware

PreFlight v2 closes the gating gap by moving from "API the agent calls" to
"middleware that executes." The middleware intercepts the action, checks the
policy, and dispatches only if the proof verifies. The agent never directly
touches the counterparty.

### What changes

| | PreFlight v1 (today) | PreFlight v2 (middleware) |
|---|---|---|
| **Who calls** | Agent calls the API | Action passes through the middleware automatically |
| **Who dispatches** | Agent dispatches the action | Middleware dispatches the action |
| **Can agent bypass?** | Yes (agent controls dispatch) | No (middleware controls dispatch) |
| **Proof binding** | Verdict bound to action text | Verdict bound to action text *and* dispatch |
| **Integration** | SDK call or HTTP request | HTTP proxy, SDK wrapper, or AgentCore Gateway interceptor |

---

## The software stack

### 1. Policy compiler (setup-time, runs once)

| | |
|---|---|
| **Input** | Natural-language policy document |
| **Process** | ARc's PMC translates to SMT-LIB. Optionally, Jolt Atlas proves the translation ($\pi_{\mathsf{SMTpolicy}}$). |
| **Output** | Committed formal policy $\bar{P}$ + cached policy proof |
| **Runs** | Once per policy update. Offline. |

Already exists in PreFlight v1. v2 adds the optional ZK proof of
compilation.

### 2. Middleware runtime (per-action, hot path)

The core new component. Five subcomponents:

**Interceptor.** Hooks into the agent's outbound call path:
- An HTTP proxy (the agent sends requests to `localhost:PORT` instead of the
  counterparty).
- An SDK wrapper (a function call that wraps the agent's dispatch).
- A Gateway interceptor Lambda in
  [AgentCore](./agentcore.md) (the managed-service path).

**Claim extractor.** Takes the action text and extracts SMT claims:
- A small classification model (sub-second, provable in Jolt Atlas).
- A small generative SLM that produces a full SMT translation (1-2s, also
  provable in Jolt Atlas).

In PreFlight v1 this happens inside the `checkIt` endpoint. In v2 it moves
into the middleware so the extraction is proved and bound to the action.

**SMT solver.** oxiz compiled to RISC-V, running inside Jolt. Checks the
extracted claims against the committed policy. Produces
$\pi_{\mathsf{SMTsolver}}$.

Already exists in PreFlight v1 (the Jolt-proved solver path).

**Proof aggregator.** Folds the extraction proof and solver proof (and
optionally the cached policy proof via recursive verification) into a single
$\pi_{\mathsf{agg}}$. See
[two-tier aggregation](../paper/proof-aggregation.md).

New in v2. PreFlight v1 produces individual proofs per `checkIt` call.

**Dispatcher.** If the aggregated proof verifies and the verdict is SAT,
dispatches the action to the counterparty together with the proof. If not,
blocks and returns an error to the agent.

New in v2. This is the enforcement point: the action fires as a consequence
of proved, policy-checked code.

### 3. Verifier (counterparty-side)

| | |
|---|---|
| **Input** | Action + proof + policy commitment |
| **Process** | Runs the Jolt / Jolt Atlas verifier (~500ms) |
| **Output** | Accept or reject |

Already exists in PreFlight v1 (`/v1/verifyProof`). v2 adds the verifier
SDK as a library for offline/embedded verification.

---

## What you'd ship

| Component | Form factor | Who runs it | v1 status |
|-----------|------------|-------------|-----------|
| Policy compiler | CLI tool or web UI | Policy owner | Exists (API) |
| Middleware runtime | Docker container, cloud service, or AgentCore interceptor | Agent operator | **New** |
| Verifier | Rust library, npm package, or Solidity contract | Counterparty | Exists (API), SDK new |

---

## Dependencies

| Dependency | Role | v1 status |
|-----------|------|-----------|
| **Jolt** | RISC-V zkVM for solver proof | In production |
| **Jolt Atlas** | ONNX zkML for extraction/classification proof | Exists, not yet in PreFlight |
| **oxiz** | Rust SMT solver compiled to RISC-V | In production |
| **ARc or equivalent SLM** | Claim extraction | In production (cloud ARc) |
| **HyperKZG SRS** | Polynomial commitment scheme | In production |

---

## Implementation phases

### Phase 1: middleware wrapper around PreFlight v1

The fastest path to v2: wrap the existing PreFlight API inside a middleware
that controls dispatch.

1. The middleware intercepts the agent's outbound action.
2. It calls PreFlight's existing `/v1/checkRelevance` and `/v1/checkIt`
   endpoints.
3. If SAT + proof verified, the middleware dispatches the action.
4. If UNSAT, the middleware blocks.

This reuses all of PreFlight v1's infrastructure. The only new code is the
interceptor and dispatcher. The gap (extraction not proved, no aggregation)
remains but execution binding is achieved.

### Phase 2: proved extraction

Move claim extraction into the middleware and prove it via Jolt Atlas.
The middleware now produces $\pi_{\mathsf{SMTplan}}$ + $\pi_{\mathsf{SMTsolver}}$,
folded into $\pi_{\mathsf{agg}}$.

**What this adds:** cryptographic binding between the action text and the SMT
claims.

**Proving overhead:** ~3-4s per action (extraction + solver).

### Phase 3: policy proof integration

Add the cached policy proof ($\pi_{\mathsf{SMTpolicy}}$) via Tier 2
recursive verification. The aggregated proof now covers the full pipeline
from natural-language policy to solver verdict.

**What this adds:** the verifier knows the formal policy came from the
declared natural-language policy, not a substituted one.

### Phase 4: full LLM wrapping

The middleware wraps the planner LLM and proves $\pi_{\mathsf{NLplan}}$. The
agent's action plan is bound to a specific prompt and model execution.

**When this makes sense:** when the planner is a small model (sub-10s proving)
or when hardware acceleration makes large-model proving practical.

---

## Architecture diagram

```
                    ┌─────────────────────────────────────────┐
                    │         PREFLIGHT v2 MIDDLEWARE          │
                    │                                         │
Agent ──(action)──► │  Interceptor                            │
                    │      │                                  │
                    │      ▼                                  │
                    │  Claim Extractor ──── π_SMTplan (Ph.2+) │
                    │      │                                  │
                    │      ▼                                  │
                    │  oxiz (in Jolt) ──── π_SMTsolver        │
                    │      │                                  │
                    │      ▼                                  │
                    │  Proof Aggregator                       │
                    │      │  ┌──────────────────────┐        │
                    │      │  │ π_SMTpolicy (cached)  │(Ph.3+)│
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
   PreFlight v1 uses `proof_id` references with download endpoints. v2 may
   need inline proofs for latency-sensitive dispatch.

2. **SRS distribution.** The HyperKZG SRS must be available to both prover
   and verifier. Currently managed internally by PreFlight. For the verifier
   SDK, it needs to be distributable.

3. **Policy commitment registry.** The counterparty needs to know what
   $\bar{P}$ to expect. PreFlight v1 uses `policy_id`. v2 may need an
   on-chain or off-chain commitment anchor for trustless verification.

4. **Reproducible builds.** The trust-your-own-setup principle requires a
   deterministic build pipeline for the RISC-V binary commitment.

5. **Multi-policy support.** An agent may need to satisfy multiple policies
   (internal compliance + counterparty requirements). Each produces a separate
   proof; the aggregator must fold all of them.
