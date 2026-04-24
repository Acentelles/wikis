# zk(SMT) middleware

The simplest middleware: the agent proves the solver step, the middleware
verifies and **executes** the action. The middleware is not just a gate; it
is the dispatch point. If the proof does not verify, the action never fires.

## Flow

```
Agent                              Middleware                    Counterparty
  │                                    │                              │
  │  1. Has policy P_SMT               │                              │
  │     (committed as P̄)               │                              │
  │                                    │                              │
  │  2. Wants to execute action A      │                              │
  │                                    │                              │
  │  3. Extracts claims C_SMT from A   │                              │
  │                                    │                              │
  │  4. Runs Oaksive inside Jolt:      │                              │
  │     Solve(P_SMT ∧ C_SMT) = b      │                              │
  │     Produces proof π               │                              │
  │                                    │                              │
  │  5. Sends (A, b, π, P̄)  ────────► │                              │
  │                                    │                              │
  │                                    │  6. Runs Jolt verifier:      │
  │                                    │     V(π, C_SMT, P̄, b) = ?   │
  │                                    │                              │
  │                                    │  7. If b = SAT and           │
  │                                    │     proof verifies:          │
  │                                    │     dispatch A  ────────────►│
  │                                    │                              │
  │                                    │  8. If not: block.           │
  │                                    │                              │
```

## What the middleware checks

1. The proof $\pi$ verifies against the public inputs $(C_{\mathsf{SMT}}, \bar{P}, b)$.
2. The verdict $b = \mathsf{SAT}$ (the action is policy-compliant).
3. The claims $C_{\mathsf{SMT}}$ correspond to the action $A$ (this is where
   $\pi_{\mathsf{SMTplan}}$ adds value; without it, the extraction is trusted).

## Who does what

| Party | Computation | Cost |
|-------|------------|------|
| Agent | SMT solving + Jolt proving | ~2s for 50 rules |
| Middleware | Jolt verification | ~500ms |
| Counterparty | Checks proof arrived with action | Negligible |

## Claim extraction subtlety

Who extracts $C_{\mathsf{SMT}}$ from $A$? If the agent does it without a
proof, the agent could extract favourable claims that don't match the action.
The policy itself provides some defence (incorrect claims may lead to UNSAT),
but a malicious agent could craft claims that are technically SAT but don't
reflect the action's semantics.

**v1 (pragmatic):** accept that the extraction is trusted. The policy catches
most mismatches via SAT/UNSAT.

**v2 (with $\pi_{\mathsf{SMTplan}}$):** the agent also proves the extraction
step via Jolt Atlas (a small SLM). The middleware checks both proofs. This
closes the extraction trust gap.

## What to build

1. A Jolt verifier library callable from the middleware (Rust, exposed via
   FFI or HTTP).
2. A serialisation format for $(A, b, \pi, \bar{P})$ that the agent sends.
3. A dispatch gate: if verification passes, forward $A$ to the counterparty;
   otherwise return an error.
4. Policy commitment management: the middleware stores $\bar{P}$ (or receives
   it from the agent per-request) and checks it against a known commitment
   registry.

## Extensions

- **Batching.** For high-throughput deployments, the middleware could
  accumulate multiple $(A_i, \pi_i)$ pairs and verify them in a batch via
  folding, amortising verification cost.
- **Logging.** The middleware logs $(A, \bar{P}, b, \pi)$ for audit trails.
  The proofs are self-contained: an auditor can re-verify at any time.
- **Multi-policy.** The agent may need to satisfy multiple policies (internal
  + external). Each produces a separate proof; the middleware checks all.
