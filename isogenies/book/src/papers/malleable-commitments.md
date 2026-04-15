# Malleable Commitments and ZK from Isogenies

> Mingjie Chen, Yi-Fu Lai, Abel Laval, Laurane Marco, Christophe Petit
> *Malleable Commitments from Group Actions and Zero-Knowledge Proofs for Circuits based on Isogenies*
> ePrint [2023/1710](https://eprint.iacr.org/2023/1710)

The PDF and full markdown of the paper live in
`raw/isogenies/2023/1710-Malleable-Commitments-from-Group-Actions-and-Zero-Knowledge-Proofs-for-Circuits-based-on-Isogenies/`.

## TL;DR

The paper gives the first generic zero-knowledge proof system for NP from
isogeny-style group-action assumptions that improves on the naive
"ZK-from-OWF" construction of Goldreich–Micali–Wigderson. The technical lever
is a new primitive — **malleable commitments** — that sits strictly between
plain (one-way-function-based) and homomorphic commitments, and that *can*
be instantiated from a known-order effective group action. From this primitive
the authors build NIZKs for arithmetic circuits, R1CS, and matrix branching
programs.

The resulting proofs are **linear in the circuit size** — so this is a
*feasibility* result, not a competitor to lattice SNARKs — but they enjoy
several pleasant properties: post-quantum security, statistical
zero-knowledge, no trusted setup, and online (hence simulation) extractability.

## Why this paper exists

The classical pipeline for building practical ZK for NP is:

1. Pick a homomorphic commitment scheme (Pedersen, ElGamal, KZG, …).
2. Use the homomorphism to glue together commitments to wire values across an
   arithmetic circuit, then run a sigma protocol on the algebraic relations.

In a post-quantum setting, this pipeline only really survives in the **lattice**
world, where module-SIS / module-LWE give some homomorphic structure. In the
**isogeny** world, there is no group structure on the set of curves at all —
only a group acting *on* it — and so a homomorphic commitment over isogenies
seems implausible. Existing isogeny-based proof systems either prove very
specific algebraic statements (signatures, ring signatures, group signatures)
or fall back to GMW-style ZK from one-way functions, which is asymptotically
fine but practically dead.

The question the paper tackles is therefore:

> Can we design a *generic* zero-knowledge proof system for NP from isogeny
> assumptions that beats the naive OWF construction?

## Main idea: relax homomorphism to malleability

The key conceptual move is to weaken what we ask of a commitment scheme.

A **malleable commitment** for a relation $R$ on the message space $M$ is a
commitment scheme together with a PPT algorithm $A^R_m : C \to C$ such that
whenever $\mathsf{com} = \mathsf{Commit}(m, r)$, the output
$\mathsf{com}' = A^R_m(\mathsf{com})$ is itself a valid commitment to some
$m'$ with $R(m, m')$. Crucially, the algorithm need not know $m$, $m'$, or the
fresh randomness $r'$ — but in this paper's instantiation, anyone holding
$(m, r)$ can recover the new opening $(m', r')$.

Homomorphic commitments are the special case where $M$, $R$, $C$ are all groups
and the malleability is the group operation. Malleable commitments only
require enough structure on the *messages* and the *randomness* — the
commitment space $C$ can stay a featureless set, which is exactly the regime
that isogenies live in.

### AGAMC: the structural sweet spot

To make this concrete, the paper specialises malleable commitments to what
they call an **Admissible Group-Action-based Malleable Commitment (AGAMC)**.
Public parameters are
$$
\mathsf{pp} = (M, R, C, X, \star)
$$
where $M$ and $R$ are abelian groups (with identities $0_M, 0_R$), $X \in C$
is a distinguished element, and
$$
\star : (M \times R) \times C \to C
$$
is a *regular*, polynomial-time group action with
$X = \mathsf{Commit}(0_M, 0_R)$. Commitments are then defined as
$$
\mathsf{Commit}(m, r) := (m, r) \star X,
$$
and the malleability is exactly the action of $(m', r')$:
$$
(m', r') \star \mathsf{Commit}(m, r) = \mathsf{Commit}(m + m', r + r').
$$
The paper requires a few standard conditions on top: efficient sampling from
subgroups of $M$ and $R$, efficient membership tests for $C$, and unique
encodings everywhere.

This is the structural contract: **the messages and the randomness sit in
groups, the commitments sit in a set acted upon by those groups, and the
malleability is exactly the action.** Notice that the commitment space itself
is *not* a group — that's the entire point.

A nice closure property: AGAMCs are closed under direct product, and one can
extend $M$ and $R$ to vector or matrix spaces. This is what later lets the
construction handle linear maps and matrix branching programs.

## Construction from a KO-EGA

Given a [known-order effective group action](../appendix/group-actions.md)
$(G, E, \star, E_0)$, fix some $s \in G$ with $s \neq 0$ and let $E = s \star E_0$.
Define an AGAMC by

- $M = R = G$, $C = E \times E$, $X = (E_0, E)$;
- $(m, r) \star' (Y_0, Y) := (r \star Y_0,\; (r + m) \star Y)$.

A commitment to $(m, r)$ is therefore the pair
$\mathsf{Commit}(m, r) = (r \star E_0,\; (r + m) \star E)$.
This is exactly the ElGamal-style ciphertext from the
[Beullens–Dobson–Katsumata–Lai–Pintore group-signature paper](https://eprint.iacr.org/2021/1366),
re-cast as a commitment.

**Security.**

- *Hiding* reduces tightly to the group-action DDH problem on $(G, E, E_0, \star)$:
  given a DDH instance $(E_1, E_2, E_3)$, set $X = (E_0, E_1)$ and answer the
  hiding adversary's challenge with $(E_2, m_b \star E_3)$. If $E_3$ is the
  real DDH element, the answer is a real commitment to $m_b$; if $E_3$ is
  uniform, the answer is information-theoretically independent of $b$.
- *Binding* is **perfect**, and follows immediately from the regularity of
  the action: if $(m_0, r_0) \star' X = (m_1, r_1) \star' X$, regularity
  forces $r_0 = r_1$ and $m_0 = m_1$.

So under DDH the commitment is computationally hiding and unconditionally
binding, in both classical and post-quantum settings.

## Sigma protocols for an AGAMC

Everything that follows is built on two sigma protocols that prove the
following relation, parameterised by a subset $M' \subseteq M$:
$$
R_{\mathsf{pp}, M'} = \{ (\mathsf{st} = c,\; \mathsf{wt} = (m, r)) \;\mid\; c = (m, r) \star X \;\wedge\; m \in M' \}.
$$
That is, "I know an opening of $c$ to some message lying in the allowed set
$M'$." The two protocols differ in what structure they assume on $M'$.

### Protocol A — small message space (Fig. 1 of the paper)

When $M' = \{m_1, \dots, m_t\}$ is *polynomially small* but otherwise has no
structure, the prover does the following.

1. Sample fresh randomness $r'$ and per-leaf labels $\mathsf{bits}_i$.
2. For each candidate $m_i \in M'$, compute
   $\mathsf{com}_i = (-m_i, r') \star u$, hash it to a leaf
   $C_i = H(\text{COM} \,\Vert\, \mathsf{com}_i \,\Vert\, \mathsf{bits}_i)$.
3. Build an **index-hiding Merkle tree** over $C_1, \dots, C_t$ — the children
   at each internal node are concatenated in alphabetical order, so revealing a
   path leaks no positional information about the leaf — and send the root.

The challenge bit decides one of two things:

- $c = 0$: prover reveals the seed used to derive everything, verifier rebuilds
  the tree from scratch and checks the root.
- $c = 1$: prover reveals $(0, r' + r)$, the path to leaf $I$ corresponding to
  the *true* message $m_I$, and the bits used at that leaf. The verifier
  reconstructs $\widetilde{\mathsf{com}} = (0, r' + r) \star X$ (which equals
  $\mathsf{com}_I$ because $\mathsf{com}_I = (-m_I, r') \star u$ and
  $u = (m_I, r) \star X$) and checks the Merkle path.

A 2-special-soundness extractor uses the two responses to recover
$(m_I, r'' - r')$, an opening of $u$ to a message in $M'$. Statistical HVZK
follows from the index-hiding property of the Merkle tree and the uniform
distribution of $r' + r$.

The proof size is $\lambda + (y + \log t \cdot \lambda + \lambda)/2$ bits per
repetition (where $y$ bounds randomness encoding and $\lambda$ is the security
parameter), and the prover does $O(t)$ malleability operations. **Logarithmic
in $|M'|$** but linear in $|M'|$ in computation, which is what forces $M'$ to
stay small in the circuit-style proofs further down.

### Protocol B — message space with subgroup structure (Fig. 2)

If $M' \leq M$ is a *subgroup*, no enumeration is needed. The prover

1. Samples a fresh $(m', r') \in M' \times R$, sends a hash commitment to
   $(m', r') \star u$.
2. On challenge $c = 0$, reveals the seed.
3. On challenge $c = 1$, reveals $(m + m', r + r')$. Because $M'$ is a
   subgroup, $m + m' \in M'$, so the verifier can re-derive a valid leaf
   $\widetilde{C} = H(\text{COM} \,\Vert\, (m + m', r + r') \star X)$ and check
   it against the original commitment.

This needs only **2 malleability operations** and proof size
$\lambda + (x + y)/2$ bits, where $x$ bounds message encoding. The
improvement over Protocol A is dramatic — a factor $\log_2(|M'|^2)$ in size and
a factor of $|M'|$ in computation — and is what makes some of the downstream
proofs practical.

### Fiat–Shamir

Both protocols are run in $\lambda$-fold parallel and Fiat-Shamir-compiled
into NIZKs. The paper notes that both NIZKs are not just simulation-extractable
but **online**-extractable, by reusing the proof technique from
[Beullens–Dobson–Katsumata–Lai–Pintore](https://eprint.iacr.org/2021/1366):
because the challenge bit is binary and the round-1 randomness is generated by
querying a PRNG (modeled as a random oracle), an extractor can read off the
$c=0$ response from the random-oracle queries regardless of which challenge
the adversary actually answers.

## Proof systems for NP statements

Three flavours of NP statement are handled. The trade-off in each is whether
the underlying ring can be embedded into a *subgroup* of $M$ (in which case
Protocol B applies) or only into an arbitrary subset (Protocol A, with the
$|M'|$ restriction).

### Arithmetic circuits over a small ring

For each gate with inputs $m_1, m_2$ and output $m_3$, the prover commits to
$(m_1, m_2, m_3)$ via the *commitment product* AGAMC and proves, using
Protocol A on the message space
$$
M'' \subseteq M'^3
$$
of valid $(\text{in}_1, \text{in}_2, \text{out})$ tuples, that the triple
satisfies the gate. For an **addition** gate the valid triples form a
subgroup of $M^3$ and Protocol B applies; for a **multiplication** gate they do
not (multiplication is not closed under the additive group structure of $M$),
so Protocol A is forced and the prover does $O(|M'|^3)$ malleability operations
per gate. This is what restricts the underlying ring to be small: the cost is
$O(|M'|^3)$ per multiplication gate, so $|M'|$ must stay polynomially bounded.

### R1CS over a small ring

For the relation $(Az) \circ (Bz) = Cz$ with $A, B, C \in F^{m \times (n+1)}$:

1. Commit to $z$, $Az$, $Bz$, $Cz$ to obtain $C_z, C_A, C_B, C_C$.
2. **Rowcheck** (consistency of $C_A, C_B, C_C$ with $C_z$ under the linear
   maps $A, B, C$). The set
   $\{(m, Am, Bm, Cm) \mid m \in M'\}$ is a subgroup of $M^4$ when $M'$ is a
   subgroup of $M$, so **Protocol B** applies and the rowcheck is cheap.
3. **Lincheck** (the Hadamard product $y_A \circ y_B = y_C$). The set of
   valid triples is *not* a subgroup, so the lincheck falls back to **Protocol
   A** and again the field has to be small.

Concretely, with $\lambda = 128$ and a hypothetical CSIDH-2048 instantiation,
the rowcheck proof is $\frac{16}{\lambda \rho}\!\left(\lambda + (3m + n + 1)\log_2 |F| + \gamma(3m + n + 1)\right)$ bytes
and the lincheck is $\frac{16}{\lambda \rho}\!\left(\lambda + \eta(m + \lambda \log_2 |F|)\right)$ bytes,
where $\rho$ is the density of $A, B, C$ and $\eta, \gamma$ are encoding sizes.

### Matrix branching programs (the workaround)

The interesting result is that **branching programs avoid the small-ring
restriction entirely**. By Barrington's theorem, any depth-$d$ fan-in-2 circuit
becomes a width-$5$ matrix branching program of length at most $4^d$. Inputs
arrive bit by bit, and at each step the running matrix is updated by left
multiplication with one of two public matrices $A_{j,0}, A_{j,1}$, depending
on the input bit. The output is "accept" iff the final product is the
identity.

The prover commits to the running matrix $M_j$ at every step and needs to
prove, for each step, that
$$
M_{j+1} = A_{j, b} M_j \quad \text{for some hidden bit } b \in \{0, 1\}.
$$
Naively this is an OR statement, and a standard OR-proof would need to provide
*all* possible statement combinations up front, which blows up exponentially
in the depth $d$. To avoid that, the paper designs a custom 3-move protocol
(Fig. 4 in the paper) that achieves the same goal with **linear** growth in $d$:

- Round 1: the prover commits to *three* objects — the two simulated
  transformations $(M', A_0 M', R')$ and $(M', A_1 M', R')$, plus a
  "real" object $(M_1 + M', A_{1-b}(M_1 + M'), R + R')$ that hides $b$ — and
  hashes them into a 2-leaf index-hiding Merkle tree whose ordering is
  permuted by $b$.
- Round 2: the verifier sends a single challenge bit.
- Round 3: depending on the challenge, the prover either reveals
  $(M_1 + M', R + R')$ together with two of the leaf labels, or reveals the
  seed used to generate the simulated leaves.

The Merkle alphabetisation means the verifier always reconstructs the same
root regardless of $b$, so the bit is hidden; soundness comes from the fact
that to answer both challenges convincingly the prover needs both sides of an
opening, which forces $M_2 = A_b M_1$ for one of the two choices.

The relation being proved is *affine* in the matrices, and affine relations
form a subgroup, so the multiplication-gate bottleneck never appears. The
result: $O(d \lambda w^2)$ KO-EGA group actions for both prover and verifier,
and the proof is **independent of the ring size** $N$. This is the only one of
the three constructions that works without restricting $|F|$.

## Properties and what this gives you

Across all three constructions, the resulting NIZKs satisfy:

- **Post-quantum** security under group-action DDH (so isogeny instantiable
  from CSI-FiSh / SCALLOP).
- **No trusted setup.**
- **Statistical** (not just computational) zero-knowledge.
- **Online extractability** ⇒ simulation extractability — i.e. proofs are
  *non-malleable*: an adversary that has seen any number of simulated proofs
  cannot produce a new accepting proof for a statement they don't have a
  witness for.
- **Linear** prover and proof size in the circuit / R1CS / branching-program
  description.

The first generic post-quantum ZK proof system from isogeny assumptions, beating
the naive ZK-from-OWF baseline.

## Limitations

The authors are upfront about the costs.

- **Multiplication gates are the bottleneck** for circuits and R1CS, because
  the set of valid $(m_1, m_2, m_1 m_2)$ triples is not a subgroup. This forces
  Protocol A and an $O(|M'|^3)$ per-gate cost, hence the small-ring restriction.
  Branching programs sidestep this entirely.
- **No vector / batch commitment.** A Pedersen-style commitment to a length-$n$
  vector with a single group element seems out of reach: a naive analog is
  *not* quantumly binding, because the relations between the basis elements
  $g_i$ can be recovered with a quantum period-finding algorithm. This blocks
  many compression tricks that the discrete-log world takes for granted.
- **Folding does not transfer.** The Bulletproofs / inner-product-argument
  folding technique relies on the homomorphism, not just on malleability, and
  the authors leave a folding-style construction over malleable commitments as
  an open problem. Without folding, the proofs are linear, never sublinear.
- **Mostly theoretical.** The only known KO-EGA instantiations
  (CSI-FiSh, SCALLOP) achieve quantum security somewhat below NIST level 1.
  The paper is honest that it should be read as a feasibility result.

## Where this fits

This is a structural result rather than a practical one: it pins down the
*minimal* algebraic interface — a regular group action on a set, with
group-structured messages and randomness — that suffices to bootstrap generic
NP zero-knowledge from isogenies, and shows how far that interface can take
you.

The really interesting open questions, in my reading, are the three concrete
limitations above. A way to handle multiplications without enumerating the
gate, a quantum-binding vector commitment, and a malleability-friendly folding
scheme would each be substantial advances on their own — and any one of them
would push isogeny-based generic ZK out of the "feasibility" bucket and into
the "competitive with lattice constructions" bucket.
