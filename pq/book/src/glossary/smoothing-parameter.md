# Smoothing parameter

The **smoothing parameter** $\eta_\varepsilon(\Lambda)$ of a lattice
$\Lambda$ is the smallest $s > 0$ such that the Gaussian mass on non-zero
dual vectors is at most $\varepsilon$:

$$\eta_\varepsilon(\Lambda) = \min \{ s > 0 : \rho_{1/s}(\Lambda^* \setminus \{0\}) \leq \varepsilon \}$$

where $\rho_\sigma(\mathbf{x}) = e^{-\pi \|\mathbf{x}\|^2 / \sigma^2}$.

When $\sigma \geq \eta_\varepsilon(\Lambda)$, a discrete Gaussian of
width $\sigma$ over $\Lambda$ is statistically close to uniform on the
quotient $\mathbb{R}^n / \Lambda$. This is the condition that makes
lattice-based hiding arguments work: the blinding randomness is "smooth
enough" to drown out the message.

In [ABBA](../papers/abba.md), hiding (Theorem 7) requires
$\sigma \geq \eta_\varepsilon(\Lambda)$ for the quaternion order
$\Lambda = \mathcal{O}_K \oplus u \mathcal{O}_K$, ensuring that the
Gaussian blinding $r \leftarrow D_{\Lambda^m, \sigma}$ hides the binary
message $\mu$.
