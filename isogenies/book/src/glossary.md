# Glossary

Short definitions for terms that recur across the notes. Entries are kept
deliberately terse; follow the links into individual papers for proper
treatments.

## Unimodular matrix

A square integer matrix $U \in \mathbb{Z}^{n \times n}$ with
$\det(U) \in \{-1, +1\}$. Equivalently, $U \in \mathrm{GL}_n(\mathbb{Z})$,
i.e., $U$ is invertible and its inverse also has integer entries.

Unimodular matrices are exactly the $\mathbb{Z}$-linear change-of-basis
transformations on the lattice $\mathbb{Z}^n$. They act on lattice bases
without changing the lattice itself, and on Gram matrices by
$Q \mapsto U^\top Q U$, which is the central operation in the Lattice
Isomorphism Problem (see the
[LIP arithmetisation idea](./papers/lip-arithmetisation-idea.md)).

Lattice term, included here because LIP shows up in the SNARK-arithmetisation
entries on this wiki.
