# One-time signature

A **one-time signature (OTS)** scheme is a digital signature scheme
that is secure only when each key pair is used to sign **exactly one
message**. Signing a second message with the same key typically leaks
the secret key entirely, destroying unforgeability.

## Definition

An OTS consists of three algorithms:

- **KeyGen** $(1^\lambda) \to (sk, pk)$: generate a key pair.
- **Sign** $(sk, \mu) \to \sigma$: produce a signature on message $\mu$.
- **Verify** $(pk, \mu, \sigma) \to \{0, 1\}$: check the signature.

Security requires that an adversary who sees *one* signature
$\sigma$ on a message $\mu$ of their choice cannot produce a valid
signature on any $\mu' \neq \mu$ (existential unforgeability under
one chosen-message attack, EUF-1-CMA).

## Why "one-time" is a feature, not a bug

OTS schemes are building blocks, not end-user primitives. Their
restricted security model makes them simpler to construct from
minimal assumptions, and they compose into stronger primitives:

1. **Lamport signatures** (1979): the original OTS, from any one-way
   function. Publish $H(s_0^i), H(s_1^i)$ for each bit $i$ of the
   message; reveal $s_{\mu_i}^i$ to sign. Two signatures on distinct
   messages leak both preimages for at least one bit position, breaking
   the scheme.
2. **Merkle trees**: chain $2^k$ OTS key pairs into a binary tree
   of hashes, yielding a many-time signature with $O(k)$ overhead.
   This is the basis of XMSS and SPHINCS+.
3. **Blind signatures**: the
   [LNP22 template](../ideas/blind-signatures.md) uses $N$ independent
   OTS key pairs as the signer's state. Each blind signing session
   consumes one OTS key; after $N$ sessions, the signer must rotate.

## Lattice-based OTS (Lyubashevsky-Micciancio)

The LM18 OTS from SIS (Lyubashevsky, Micciancio, *Asymptotically
Efficient Lattice-Based Digital Signatures*, J. Cryptology 2018):

- **KeyGen.** Sample $\mathbf{s}, \mathbf{y} \leftarrow D_{\mathbb{Z}^m, \sigma}$
  (short Gaussians). Public key:
  $\mathbf{v} = A\mathbf{s}$, $\mathbf{w} = A\mathbf{y}$, where
  $A \in \mathbb{Z}_q^{n \times m}$ is a shared public matrix.
- **Sign** $(\mathbf{s}, \mathbf{y}, \mu)$. Output
  $\mathbf{z} = \mathbf{s}\mu + \mathbf{y}$.
- **Verify** $(pk, \mathbf{z}, \mu)$. Check
  $A\mathbf{z} = \mathbf{v}\mu + \mathbf{w}$ and
  $\|\mathbf{z}\|_\infty \leq B$.

Correctness: $A\mathbf{z} = A(\mathbf{s}\mu + \mathbf{y}) = (A\mathbf{s})\mu + A\mathbf{y} = \mathbf{v}\mu + \mathbf{w}$.

Unforgeability: a forgery $\mathbf{z}' \neq \mathbf{z}$ on the same
$\mu$ gives $A(\mathbf{z} - \mathbf{z}') = 0$ with
$\mathbf{z} - \mathbf{z}'$ short and nonzero, i.e. a SIS solution.
A forgery on a different $\mu'$ requires the entropy argument from
[LNP22 page 5](../papers/lnp22-blind-signatures.md): the reduction
either extracts $\mathbf{s}$ or a SIS solution.

**One-time-ness.** Two signatures
$\mathbf{z}_1 = \mathbf{s}\mu_1 + \mathbf{y}$ and
$\mathbf{z}_2 = \mathbf{s}\mu_2 + \mathbf{y}$ on distinct messages
yield $\mathbf{z}_1 - \mathbf{z}_2 = \mathbf{s}(\mu_1 - \mu_2)$. If
$\mu_1 - \mu_2$ is invertible (the generic case), the attacker
recovers $\mathbf{s}$ and then $\mathbf{y}$, forging on any message.

## ComSIS-OTS (from ABBA)

The [ABBA paper](../papers/abba.md) (Section 8) constructs a
commutator-based analogue; see the
[ComSIS-OTS idea page](../ideas/comssis-ots.md) for the full scheme
and analysis.
