# Transference theorem

**Banaszczyk's transference theorem** (1993) bounds the product of the
shortest vectors in a lattice and its [dual](./dual-lattice.md):

$$\lambda_1(\Lambda) \cdot \lambda_1(\Lambda^*) \leq n$$

where $n$ is the rank. More generally,
$\lambda_i(\Lambda) \cdot \lambda_{n+1-i}(\Lambda^*) \leq n$ for all
$i \in [n]$.

This captures the fundamental tension in lattice cryptography:

- **Security** needs short vectors to be hard to find
  ($\lambda_1(\Lambda)$ large).
- **Correctness** of decryption/verification typically requires short
  dual vectors ($\lambda_1(\Lambda^*)$ small, which means the noise
  budget fits within the fundamental domain).

The transference bound says you cannot have both $\Lambda$ and
$\Lambda^*$ simultaneously "well-spread"; making one lattice harder
to solve makes the other easier, and this ratio is at most a factor of
$n$.
