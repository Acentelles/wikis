# Sum-Check over the Monomial Basis

| | |
|---|---|
| **Authors** | Quang Dao, Ari Biswas, Liam Eagen, Andrew Milson, Shahar Papini, Justin Thaler |
| **eprint** | 2026/762 |

## Summary

A projective variant of the sum-check protocol obtained by changing the interpolating set from the Boolean hypercube $\{0,1\}^n$ to the **infinity hypercube** $\{0, \infty\}^n$. Evaluating a multilinear polynomial at a point in $\{0,\infty\}^n$ directly extracts its corresponding monomial coefficient. This yields a $\sim$10% end-to-end speedup.

## Key Idea: Projective Binding

In standard sum-check, "binding" a variable $x_i$ at a random challenge $r$ requires computing

$$f(r, x_{i+1}, \ldots) = (1-r) \cdot f(0, x_{i+1}, \ldots) + r \cdot f(1, x_{i+1}, \ldots),$$

which involves a subtraction (the $(1-r)$ term). In the projective variant, binding becomes

$$f(r, x_{i+1}, \ldots) = f|_{x_i=0}(\ldots) + r \cdot f|_{x_i=\infty}(\ldots),$$

eliminating all field subtractions. Since subtractions and additions have the same cost in most fields, the savings come from the fact that the monomial form simplifies the binding formula and aligns more naturally with the commitment scheme.

## Specific Optimisations

- **Subtraction-free binding.** The projective binding formula eliminates all field subtractions from the sum-check prover's inner loop, replacing them with additions.

- **Structured polynomial speedups.** For the equality polynomial and the "less-than" polynomial (used in memory checking), the projective interpolants admit evaluation procedures with fewer field operations than their Boolean-hypercube counterparts.

- **PCS alignment.** Monomial-coefficient form aligns naturally with WHIR and other commitment schemes that operate on coefficient vectors, removing a basis mismatch.

## Challenge-Subset Optimisation

For sum-check over $\sim$256-bit prime fields targeting $\sim$128 bits of security, it suffices to sample challenges from a subset of size $\sim 2^{128}$. By choosing this subset to correspond to upper-limb values in Montgomery form, field multiplication becomes cheaper (the lower limbs are zero). Combined with projective binding, this yields a further speedup for sum-check binding.

## Results

- $\sim$10% end-to-end speedup on BN254 and on a pseudo-Mersenne 128-bit prime field.
- Near-drop-in replacement: requires only local changes to polynomial representations, round identities, and evaluation formulas.
