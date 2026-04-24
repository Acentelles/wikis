# SIS/LWE duality

The Short Integer Solution (SIS) problem and Learning With Errors (LWE)
problem are dual formulations of the same underlying lattice hardness:

- **SIS** (primal): given a random matrix
  $A \in \mathbb{Z}_q^{n \times m}$, find a short nonzero
  $\mathbf{z} \in \mathbb{Z}^m$ with $A\mathbf{z} = 0 \bmod q$. This is
  finding a short vector in the $q$-ary kernel lattice
  $\Lambda^\perp(A) = \{ \mathbf{z} : A\mathbf{z} \equiv 0 \}$.

- **LWE** (dual): given $(A, A^T\mathbf{s} + \mathbf{e})$, recover
  $\mathbf{s}$ (or distinguish from uniform). This is a closest-vector
  problem on the [dual lattice](./dual-lattice.md) (the image lattice
  of $A^T$).

Regev's reduction (2005) connects worst-case GapSVP on $\Lambda$ to
average-case LWE on (a lattice related to) $\Lambda^*$, with the
[smoothing parameter](./smoothing-parameter.md) as the bridge: the
reduction samples Gaussians wide enough to smooth the dual lattice,
turning worst-case short-vector instances into LWE samples.

In module/ring variants (Module-SIS, Module-LWE), the same duality
holds over the ring $R_q$. The commitment schemes in this wiki
([ABBA](../papers/abba.md), Ajtai in [Hachi](../papers/hachi.md)) are
SIS-based (binding = hardness of finding short kernel elements); their
hiding arguments rely on the dual (LWE/smoothing) side.
