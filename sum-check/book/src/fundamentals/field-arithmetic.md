# Field Arithmetic: Native, Non-Native, and Extension Fields

## The Two Fields of an Elliptic Curve

An elliptic curve $E/\mathbb{F}_q$ carries two distinct fields:

- **Base field** $\mathbb{F}_q$. The field over which the curve equation $y^2 = x^3 + ax + b$ is defined. Point coordinates $(x, y)$ live here.
- **Scalar field** $\mathbb{F}_r$. The field of scalars that multiply curve points, where $r = |E(\mathbb{F}_q)|$ is the group order. When we write $[k]P$, the scalar $k$ lives in $\mathbb{F}_r$.

For BN254, $r \approx q$ (both $\sim$254 bits), but they are different primes.

## Native vs. Non-Native Arithmetic

A SNARK (or a zkVM) works over a single **native field**. All witness values, constraint coefficients, and sum-check evaluations are elements of that field. For Jolt with BN254, the native field is $\mathbb{F}_r$.

**Native arithmetic** ($\mathbb{F}_r$ operations inside an $\mathbb{F}_r$-native SNARK):
- Each field multiplication costs 1 R1CS constraint, or $\sim$1 VM cycle.
- Sum-check round messages, evaluation claims, and witness polynomials are all $\mathbb{F}_r$ elements.
- When the Efficient Recursion paper says "this is pure $\mathbb{F}_r$ arithmetic," it means: the computation involves only additions and multiplications in the SNARK's native field, with no group operations, no extension fields, and no foreign-field emulation.

**Non-native arithmetic** ($\mathbb{F}_q$ operations inside an $\mathbb{F}_r$-native SNARK):
- Since $q \neq r$, the SNARK cannot use its built-in field operations for $\mathbb{F}_q$.
- Each $\mathbb{F}_q$ multiplication must be emulated as a multi-limb integer multiplication modulo $q$, followed by range checks on the limbs.
- This costs **hundreds of R1CS constraints** (or VM cycles) per operation.
- For $\mathbb{F}_{q^{12}}$ (the pairing target group), the situation is roughly $12^2 = 144$ times worse again.

This asymmetry is exactly why naive Jolt recursion costs billions of VM cycles: the Dory verifier's $G_T$ exponentiations require $\mathbb{F}_{q^{12}}$ arithmetic, all of which must be emulated non-natively.

## Extension Fields

A degree-$k$ extension field $\mathbb{F}_{q^k}$ is constructed as the quotient ring

$$\mathbb{F}_{q^k} = \mathbb{F}_q[X] / p(X),$$

where $p(X) \in \mathbb{F}_q[X]$ is an irreducible polynomial of degree $k$. Each element $a \in \mathbb{F}_{q^k}$ corresponds to a polynomial $a(X) \in \mathbb{F}_q[X]$ of degree at most $k - 1$, and multiplication is polynomial multiplication modulo $p(X)$.

The pairing target group $G_T$ lives inside $\mathbb{F}_{q^{12}}^*$ for BN254. Each $G_T$ element requires 12 base-field coefficients, and a single multiplication in $G_T$ is an order of magnitude more expensive than a base-field multiplication.

### The Polynomial Division Identity

For $a, b, c \in \mathbb{F}_{q^k}$, the equation $a \cdot b = c$ holds if and only if there exists a quotient polynomial $Q(X) \in \mathbb{F}_q[X]$ with $\deg(Q) \le k - 2$ such that

$$a(X) \cdot b(X) = c(X) + Q(X) \cdot p(X).$$

This is a polynomial identity over $\mathbb{F}_q$: verifying an extension-field multiplication reduces to checking that four polynomials (all with $\mathbb{F}_q$ coefficients) satisfy this relation, with $Q$ provided as an additional witness.

**Example ($G_T \subset \mathbb{F}_{q^{12}}$).** To verify $a \cdot b = c$ in $G_T$, check that there exists $Q(X)$ with $\deg(Q) \le 10$ such that $a(X) \cdot b(X) = c(X) + Q(X) \cdot p(X)$, where $p(X)$ is the degree-12 irreducible. Inside a SNARK native to $\mathbb{F}_q$, each coefficient comparison is a single field operation.

The representation of group elements as MLEs and the PIOPs for proving group operations are covered in [Group Arithmetic](./group-arithmetic.md).

## Curve Cycles and Why They Eliminate Non-Native Arithmetic

A **2-cycle** of elliptic curves consists of two curves whose scalar and base fields are swapped:

| Curve | Base field | Scalar field |
|---|---|---|
| BN254 | $\mathbb{F}_q$ | $\mathbb{F}_r$ |
| Grumpkin | $\mathbb{F}_r$ | $\mathbb{F}_q$ |

A SNARK over BN254 is native to $\mathbb{F}_r$. A SNARK over Grumpkin is native to $\mathbb{F}_q$. This is why the Efficient Recursion paper uses both:

1. The **main Jolt SNARK** operates over $\mathbb{F}_r$ (BN254's scalar field). Sum-check, witness polynomials, and evaluation claims are all $\mathbb{F}_r$ elements.

2. The **Dory verifier** performs group operations with coordinates in $\mathbb{F}_q$ (BN254's base field). Proving these inside the $\mathbb{F}_r$-native SNARK would require expensive non-native emulation.

3. The **auxiliary SNARK** ($\pi'$) uses Hyrax over Grumpkin, where $\mathbb{F}_q$ is the native field. The Dory verification steps (point additions, scalar multiplications, $G_T$ exponentiations) all involve $\mathbb{F}_q$ coordinates, so they become native arithmetic in this SNARK.

4. The **verifier of $\pi'$** performs Grumpkin $G_1$ operations, whose coordinates are in $\mathbb{F}_r$. This is native to the main Jolt SNARK.

The result: no non-native arithmetic anywhere in the recursion path.

## Cost Summary

| Operation | Inside $\mathbb{F}_r$-native SNARK | Inside $\mathbb{F}_q$-native SNARK |
|---|---|---|
| $\mathbb{F}_r$ multiplication | 1 constraint (native) | ~hundreds (non-native) |
| $\mathbb{F}_q$ multiplication | ~hundreds (non-native) | 1 constraint (native) |
| $\mathbb{F}_{q^{12}}$ multiplication | ~tens of thousands | ~12 constraints (via poly. division identity) |
| BN254 $G_1$ scalar mul | coordinates in $\mathbb{F}_q$, non-native | coordinates in $\mathbb{F}_q$, native |
| Grumpkin $G_1$ scalar mul | coordinates in $\mathbb{F}_r$, native | coordinates in $\mathbb{F}_r$, non-native |

(*Proofs, Arguments, and Zero-Knowledge*, Chapters 18 and 19; Efficient Recursion for the Jolt zkVM, Sections 2.4-2.6)
