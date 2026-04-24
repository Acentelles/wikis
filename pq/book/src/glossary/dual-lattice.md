# Dual lattice

The **dual lattice** $\Lambda^*$ of a lattice $\Lambda \subset \mathbb{R}^n$
is the set of all vectors whose inner product with every lattice vector
is an integer:

$$\Lambda^* = \{ \mathbf{x} \in \mathbb{R}^n : \langle \mathbf{x}, \mathbf{v} \rangle \in \mathbb{Z} \text{ for all } \mathbf{v} \in \Lambda \}$$

If $\Lambda$ has basis $B$, the dual has basis $B^{-T}$, and the
determinants are reciprocal: $\det(\Lambda^*) = 1/\det(\Lambda)$.

**Geometric intuition.** The dual "inverts" the geometry: short
directions in $\Lambda$ become long directions in $\Lambda^*$, and vice
versa. This is formalised by Banaszczyk's
[transference theorem](./transference-theorem.md):
$\lambda_1(\Lambda) \cdot \lambda_1(\Lambda^*) \leq n$.

**Fourier-analytic intuition.** A dual vector $\mathbf{w}$ defines the
character $e^{2\pi i \langle \mathbf{w}, \cdot \rangle}$, which equals 1
on every point of $\Lambda$. The dual lattice is exactly the set of
"frequencies" for which the lattice is invisible. The Poisson summation
formula relates sums over $\Lambda$ to sums over $\Lambda^*$:

$$\sum_{\mathbf{v} \in \Lambda} f(\mathbf{v}) = \frac{1}{\det(\Lambda)} \sum_{\mathbf{w} \in \Lambda^*} \hat{f}(\mathbf{w})$$

**For ideal lattices.** If $\Lambda$ is the ideal $I$ in a number ring
$R = \mathbb{Z}[X]/(f(X))$, the dual is the codifferent
$I^\vee = \{ x \in K : \mathrm{Tr}(x \cdot I) \subseteq \mathbb{Z} \}$.

The dual lattice appears throughout lattice cryptography:
[SIS/LWE duality](./sis-lwe-duality.md),
[smoothing parameter](./smoothing-parameter.md), and
[transference theorems](./transference-theorem.md).
