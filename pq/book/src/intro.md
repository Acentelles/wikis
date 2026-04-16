# Introduction

This book collects my notes on post-quantum cryptography, with a particular
focus on constructions that build higher-level primitives (commitments,
signatures, zero-knowledge proofs, blind signatures) from the two algebraic
worlds I care about most: **isogeny-based** and **lattice-based** cryptography.

Both settings share a design tension that runs through these notes. Classical
discrete-log or pairing-based schemes exploit rich homomorphic structure
(Pedersen, Schnorr, KZG). In the post-quantum setting that structure either
disappears (the class group of an imaginary quadratic order acts on a set of
supersingular elliptic curves, but the set itself carries no group structure)
or becomes noisy and bounded (lattice commitments hide messages under small
randomness that grows under every operation). A recurring theme is finding
ways to recover *just enough* algebraic structure to build the primitive at
hand, whether that is malleability, linkability, or the ability to compress
a transcript.

## How this book is organised

- **Papers**, focused write-ups of papers I have read, summarising the main
  ideas, the technical contributions, and (where relevant) the limitations.
  Each entry should stand alone: someone reading only that page should walk
  away with the gist without needing to read the paper itself.
- **Ideas and proposals**, working notes on directions I think are worth
  pursuing but that are not (yet) papers.
- **Appendix**, shared background material referenced by the paper notes,
  including the formal model for cryptographic group actions.

## Where to start

- New to the area? → [Group Actions and KO-EGA](./appendix/group-actions.md)
- Want a survey of post-quantum ZK from isogenies? →
  [Malleable Commitments and ZK from Isogenies](./papers/malleable-commitments.md)
- Interested in lattice commitments and blind signatures? →
  [Blind signatures from commutator commitments](./ideas/blind-signatures.md)
