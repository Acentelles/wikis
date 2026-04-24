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

## Classification models: the practical sweet spot

For the zkML middleware, proving a full autoregressive LLM (many tokens,
large model) is expensive. But the middleware often does not need text
generation; it needs a *decision*: is this action compliant or not?

Classification models are a natural fit:

- They are tiny (sub-million parameters).
- They produce a single label (compliant / non-compliant / ambiguous), not a
  token sequence.
- They work with Jolt Atlas today, with sub-second proving times.
- The middleware only needs to prove the classifier ran on the action text and
  produced a verdict, not generate a full translation.

This reframes the zkML middleware: instead of wrapping a generative SLM, wrap
a binary or multi-class classifier trained on the policy domain. The
classifier's verdict is the guardrail output. The proof attests that the
committed classifier model produced that verdict on that input.

## Feasibility

| What is proved | System | Latency |
|---------------|--------|---------|
| Classification model | Jolt Atlas | Sub-second |
| Small SLM extraction ($\pi_{\mathsf{SMTplan}}$) | Jolt Atlas | 1-2s (fine-tuned SLM) |
| Full LLM inference ($\pi_{\mathsf{NLplan}}$) | Jolt Atlas | 14-38s (nanoGPT/GPT-2) |
| SMT solver ($\pi_{\mathsf{SMTsolver}}$) | Jolt | ~2s (50 rules) |

The classification model is the cheapest to prove and the most immediately
deployable. Full LLM inference is the most expensive.

## Pragmatic v1: classifier + solver

The planner LLM runs outside the middleware (unproved). Its output
$A_{\mathsf{NL}}$ is taken as input to the middleware. The middleware proves:

1. A classification model on $A_{\mathsf{NL}}$ (sub-second).
2. The solver step (~2s).
3. Optionally, agreement between the classifier's verdict and a proprietary
   model's verdict.

This gives proved classification + proved solver without the cost of proving
the full LLM or even a generative SLM. The gap (unproved planner) is exactly
what $\pi_{\mathsf{NLplan}}$ closes when the budget allows it.

**Total overhead for v1:** ~2-3s per action.

## v2: wrap extraction (SLM) + solver

Replace the classifier with a small generative SLM that produces a full
SMT translation. Proving cost increases to ~3-4s but provides richer
extraction semantics.

## v3: full wrapping

When proving cost decreases (hardware acceleration, algorithmic improvements)
or the model is small enough, the middleware proves the full pipeline including
$\pi_{\mathsf{NLplan}}$. The middleware becomes a fully-proved execution
environment.

## The middleware executes, not just gates

As discussed in the [middleware overview](./middleware.md), the middleware
must be the component that dispatches the action, not merely a gate that
approves it. If the middleware only gates, the agent retains control of
dispatch and can execute a different action.

When the middleware is small and deterministic, it can itself run inside the
zkVM. The proof then attests: "this program, which includes the dispatch call,
was executed on these inputs." The action is a side effect of proved code.

## Deployment considerations

- The middleware must be co-located with the LLM serving infrastructure (or
  *be* the serving infrastructure). It cannot run client-side if it needs
  access to the model weights.
- For hidden-model deployments, the middleware is the natural trust boundary:
  it holds the model weights, proves inference, and never exposes them to the
  counterparty.
- The middleware can serve multiple agents against the same policy, amortising
  the policy proof ($\pi_{\mathsf{SMTpolicy}}$) across all of them.
- The proving infrastructure is heavy (Jolt compiler, zkVM runtime, SRS).
  This pushes deployment toward managed services or dedicated infrastructure
  rather than lightweight client-side integration. See the
  [deployment models](./middleware.md#deployment-models) discussion.
