# Introduction

This book collects my notes on isogeny-based cryptography, with a particular
focus on constructions that build higher-level primitives (commitments,
signatures, zero-knowledge proofs) on top of cryptographic group actions.

Isogeny-based cryptography is one of the few candidate post-quantum settings
where the algebraic structure available to a designer is *much weaker* than the
classical groups used by discrete-log or pairing-based schemes. Concretely, the
class group of an imaginary quadratic order acts on a set of supersingular
elliptic curves, but the set itself carries **no group structure**. This means
techniques that rely on homomorphism — Pedersen commitments, Schnorr-style
proofs, polynomial commitments via group exponentiation — do not transfer
directly. A recurring theme in the literature, and in these notes, is finding
ways to recover *just enough* structure from a group action to build the
primitive at hand.

## How this book is organised

- **Papers** — focused write-ups of papers I have read, summarising the main
  ideas, the technical contributions, and (where relevant) the limitations.
  Each entry should stand alone: someone reading only that page should walk
  away with the gist without needing to read the paper itself.
- **Appendix** — shared background material referenced by the paper notes,
  including the formal model for cryptographic group actions.

## Where to start

- New to the area? → [Group Actions and KO-EGA](./appendix/group-actions.md)
- Want a survey of post-quantum ZK from isogenies? →
  [Malleable Commitments and ZK from Isogenies](./papers/malleable-commitments.md)
