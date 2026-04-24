# Model and input hiding

## Input hiding is negligible-cost

The overhead of hiding inputs (prompts, action text, extracted claims, policy
rules, model weights) is negligible when using the NovaBlindFold technique.
Jolt Atlas achieves ZK by deferring all sumcheck verification claims to a
single folded R1CS instance and randomising it via Nova folding. The cost is
a constant-factor overhead on the final Spartan decider, independent of the
number of hidden inputs.

**Implication:** there is little reason not to enable input/policy hiding by
default.

## Weight hiding

Weights move from public constants baked into preprocessing to prover-private
witness data bound to the transcript before any challenge is drawn. Since Jolt
Atlas already hides its witness vector via NovaBlindFold, it suffices to
declare weight MLEs as entries of that witness vector.

## Architecture hiding

Architecture hiding is harder. The Jolt Atlas verifier is indexed by the model
graph: R1CS dimensions, Fiat-Shamir challenge lengths, witness-slot layout,
operator dispatch, and lookup table sizes are all functions of the graph.
Redesigning the verifier to work without the graph would require a pervasive
IOP rewrite.

Instead, we use recursive verification: the inner prover $P_M$ produces a
standard Jolt Atlas proof $\pi_M$. A second prover $P_V$ proves that
$V(M, \pi_M, \bar{\omega}, x) = \mathsf{accept}$ with the model $M$, the
proof $\pi_M$, and all intermediate artefacts as private witnesses. The
recursive verifier $V'$ checks $\pi_V$ against only the model commitment
$\bar{M}$ and the I/O $x$.

The inner verifier is already compressed to a single folded R1CS check + PCS
opening, so the recursive circuit is compact and its size is independent of
model depth.

## Cost summary

| What is hidden | Technique | Overhead |
|---------------|-----------|----------|
| Inputs, claims, policy | NovaBlindFold | Negligible |
| Model weights | Witness promotion | Negligible |
| Model architecture | Recursive verification | Significant (one extra proof) |
