# Translation correctness

## The problem

The translation model (SLM) converts natural-language actions into SMT claims.
If the SLM runs on the agent operator's hardware, the operator could tamper
with it or fabricate translations. Running $k$ independent translators and
checking verdict agreement is not enough: without a ZKP, no translator is
proven to have run.

## The design: proven SLM + proprietary agreement

Two layers:

1. **Prove the SLM ran.** The local SLM is small enough to prove via Jolt
   Atlas. The proof $\pi_{\mathsf{SMTplan}}$ attests that a specific committed
   model was executed on the action text and produced a specific translation.
   This is the binding anchor.

2. **Check agreement with a proprietary model.** A larger proprietary
   translator (e.g., ARc's production model) runs on the provider's
   infrastructure and produces its own verdict. Inside the ZK circuit, a
   scalar equality check verifies that the proven SLM's solver verdict matches
   the proprietary verdict (supplied as a private witness).

## Why this is better than the alternatives

- Better than k-SLM agreement without ZKP: at least one model is *proven* to
  have run on the actual input.
- Better than proving only the SLM: agreement with a more capable proprietary
  model catches translation errors the small model might make.
- Practical: ZK overhead scales with the small model, not the proprietary one.
  The proprietary verdict enters the circuit as a single scalar witness.

## Residual threat

The proprietary model's verdict is taken on trust. A compromise of the
provider's infrastructure could inject a false agreement signal. But this is
narrow: the attacker must compromise the provider *and* the SLM's translation
must happen to match the injected verdict for the specific action.

## AWS collaboration point

AWS has offered to fine-tune a small model at 1-2s latency for this role. Base
model family, training data, and target accuracy are open questions.
