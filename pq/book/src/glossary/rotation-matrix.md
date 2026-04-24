# Rotation matrix

The **rotation matrix** $\mathrm{rot}(a) \in \mathbb{F}_q^{d \times d}$
of a ring element $a \in R_q = \mathbb{F}_q[X]/(\Phi_\eta)$ is the
matrix that represents multiplication by $a$ as a
$\mathbb{F}_q$-linear map on the coefficient vector. That is, for all
$b \in R_q$:

$$\mathrm{rot}(a) \cdot \mathrm{cf}(b) = \mathrm{cf}(a \cdot b)$$

where $\mathrm{cf}(b) = (b_0, b_1, \ldots, b_{d-1})^T$ is the
coefficient vector of $b$.

**Construction.** Let $\Phi_\eta = X^d + c_{d-1}X^{d-1} + \cdots + c_0$
and define the **shift matrix**:

$$F = \begin{pmatrix} 0 & & & -c_0 \\ 1 & 0 & & -c_1 \\ & \ddots & \ddots & \vdots \\ & & 1 & -c_{d-1} \end{pmatrix} \in \mathbb{F}_q^{d \times d}$$

$F$ represents multiplication by $X$ in $R_q$: $F \cdot \mathrm{cf}(b) = \mathrm{cf}(X \cdot b)$.
The rotation matrix is then:

$$\mathrm{rot}(a) = \begin{pmatrix} \mathrm{cf}(a) & F \cdot \mathrm{cf}(a) & \cdots & F^{d-1} \cdot \mathrm{cf}(a) \end{pmatrix}$$

The $j$-th column is $\mathrm{cf}(X^j \cdot a)$, i.e. the coefficients
of $a$ shifted $j$ times through the cyclotomic reduction.

**Power-of-two case.** For $\Phi_\eta = X^d + 1$ (e.g. $\eta = 128$,
$d = 64$), the shift matrix simply negates the last coefficient and
rotates: $F \cdot (a_0, \ldots, a_{d-1})^T = (-a_{d-1}, a_0, \ldots, a_{d-2})^T$.
The rotation matrix is then a **negacyclic** (skew-circulant) matrix:

$$\mathrm{rot}(a) = \begin{pmatrix} a_0 & -a_{d-1} & \cdots & -a_1 \\ a_1 & a_0 & \cdots & -a_2 \\ \vdots & \vdots & \ddots & \vdots \\ a_{d-1} & a_{d-2} & \cdots & a_0 \end{pmatrix}$$

Each column is the previous column cyclically shifted down by one, with
the wrapped entry negated. This is a structured matrix: all $d^2$
entries are determined by the $d$ coefficients of $a$.

**Why it matters.** The rotation matrix is the bridge between ring
arithmetic and $\mathbb{F}_q$-linear algebra. It converts ring
multiplications in $R_q$ into matrix-vector products over
$\mathbb{F}_q$, which is essential in several settings:

1. **Neo's sumcheck over $\mathbb{F}_q$** (Neo, Definition 7): Neo
   avoids running the sumcheck protocol over the ring $R_q$ (which would
   require extension-field challenges for soundness). Instead, it maps
   every $R_q$-relation $c = a \cdot b$ into the $\mathbb{F}_q$-linear
   relation $\mathrm{cf}(c) = \mathrm{rot}(a) \cdot \mathrm{cf}(b)$.
   The sumcheck then runs over $\mathbb{F}_q$ (or a small extension),
   and the rotation matrix structure makes the verifier's work efficient.

2. **Pay-per-bit commitment** (Neo, Section 3.2): when $b$ is a binary
   vector ($b_i \in \{0, 1\}$), the product
   $\mathrm{rot}(a) \cdot \mathrm{cf}(b)$ reduces to summing the
   columns of $\mathrm{rot}(a)$ where $b_i = 1$. Each column is the
   previous column shifted by one (the `rot_step` operation), so
   computing all $d$ columns costs $O(d)$ additions total, not $O(d^2)$.
   This is Neo's pay-per-bit property: the commitment cost scales with
   the Hamming weight of the witness, not its length.

3. **Ajtai commitment as matrix product**: the Ajtai commitment
   $c = M \cdot z$ for $M \in R_q^{\kappa \times m}$ and
   $z \in R_q^m$ is equivalent to a $\kappa d \times md$ matrix-vector
   product over $\mathbb{F}_q$, where each $d \times d$ block of the
   expanded matrix is $\mathrm{rot}(M_{i,j})$. This structured-matrix
   view is what makes lattice commitments compatible with
   $\mathbb{F}_q$-native proof systems.

4. **Coefficient-wise blind signing**
   ([LNP22](../papers/lnp22-blind-signatures.md), Section 1.2): the
   blind signature relation $A\mathbf{z} = \mathbf{v}\mu + \mathbf{w}$
   (Eq. 1) involves the ring product $\mu s_j$ between the message
   polynomial $\mu$ and each secret key coefficient $s_j$. The signer
   processes this product coefficient-by-coefficient via
   $\mathrm{cf}(\mu s_j) = \mathrm{rot}(\mu) \cdot \mathrm{cf}(s_j)$.
   This decomposition is what allows the signer to encrypt each
   coordinate of the signature independently using an LWE-based scheme
   (Eq. 2--3), without ever revealing $\mathbf{s}$ or $\mu$ to the
   other party: the ring structure of $\mu s_j$ is handled by
   $\mathrm{rot}(\mu)$, while the LWE encryption operates on individual
   $\mathbb{F}_q$-coefficients.

The ring of all rotation matrices
$\mathcal{S} = \{ \mathrm{rot}(a) : a \in R_q \} \subset \mathbb{F}_q^{d \times d}$
is a commutative subring of $d \times d$ matrices, isomorphic to $R_q$
itself. It contains all scalar matrices (corresponding to constant
polynomials $a \in \mathbb{F}_q$) as a subring (Neo, Definition 12).
