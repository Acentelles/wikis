# zkML middleware

The harder case: the middleware wraps the LLM so that every input/output is
proved.

## The self-awareness problem

In the [zk(SMT) case](./zk-smt.md), the agent produces the proof and the
middleware verifies it. But for the LLM inference step ($\pi_{\mathsf{NLplan}}$
or $\pi_{\mathsf{SMTplan}}$), the agent cannot prove its own inference after
the fact: the middleware needs to know the proof covers the *actual* inference,
not a fabricated one.

If the agent runs the LLM and then claims "this is what the model produced,"
the claim is unverified. The agent could have run a different model, or
fabricated the output entirely.

## Solution: the middleware is the execution environment

The middleware is not just a gate; it wraps the LLM:

```
Prompt Q ───► Middleware ───► LLM (inside middleware) ───► A_NL
                 │
                 ├── Prove: π_NLplan (Jolt Atlas)
                 │
                 ├── Extract claims: C_SMT from A_NL
                 │
                 ├── Prove extraction: π_SMTplan (Jolt Atlas, small SLM)
                 │
                 ├── Run solver: Solve(P_SMT ∧ C_SMT) = b
                 │
                 ├── Prove solver: π_SMTsolver (Jolt)
                 │
                 ├── If b = SAT and all proofs verify:
                 │      dispatch A_NL + π_agg ───► Counterparty
                 │
                 └── If not: block.
```

The middleware sees the raw inputs, runs the model, proves the execution, and
controls dispatch. The counterparty trusts the proof, not the middleware.

## Feasibility

| What is proved | System | Latency |
|---------------|--------|---------|
| Full LLM inference ($\pi_{\mathsf{NLplan}}$) | Jolt Atlas | 14-38s (nanoGPT/GPT-2) |
| Small SLM extraction ($\pi_{\mathsf{SMTplan}}$) | Jolt Atlas | 1-2s (fine-tuned SLM) |
| SMT solver ($\pi_{\mathsf{SMTsolver}}$) | Jolt | ~2s (50 rules) |

The full LLM proof is the bottleneck. For large production models the proving
cost may be prohibitive.

## Pragmatic v1: wrap extraction + solver only

The planner LLM runs outside the middleware (unproved). Its output
$A_{\mathsf{NL}}$ is taken as input to the middleware. The middleware proves:

1. The extraction step (small SLM, 1-2s).
2. The solver step (~2s).
3. Agreement between the SLM's verdict and the proprietary model's verdict.

This gives proved extraction + proved solver without the cost of proving the
full LLM. The gap (unproved planner) is exactly what $\pi_{\mathsf{NLplan}}$
closes when the budget allows it.

**Total overhead for v1:** ~3-4s per action.

## v2: full wrapping

When proving cost decreases (hardware acceleration, algorithmic improvements)
or the model is small enough (the fine-tuned SLM), the middleware proves the
full pipeline including $\pi_{\mathsf{NLplan}}$. The middleware becomes a
fully-proved execution environment.

## Deployment considerations

- The middleware must be co-located with the LLM serving infrastructure (or
  *be* the serving infrastructure). It cannot run client-side if it needs
  access to the model weights.
- For hidden-model deployments, the middleware is the natural trust boundary:
  it holds the model weights, proves inference, and never exposes them to the
  counterparty.
- The middleware can serve multiple agents against the same policy, amortising
  the policy proof ($\pi_{\mathsf{SMTpolicy}}$) across all of them.
