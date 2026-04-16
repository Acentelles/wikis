# Group Actions and KO-EGA

Most of the constructions in these notes are phrased in terms of an *effective*
cryptographic group action rather than directly in the language of isogenies.
This page collects the definitions used throughout the book.

## Group action

A group $(G, +)$ **acts** on a set $E$ if there is a map
$$
\star : G \times E \to E
$$
satisfying:

1. **Identity.** $0 \star E = E$ for every $E \in E$.
2. **Compatibility.** $(g + h) \star E = g \star (h \star E)$ for all $g, h \in G$ and $E \in E$.

The action is **regular** if it is transitive and free, i.e. for every
$x_1, x_2 \in E$ there is a unique $g \in G$ with $x_2 = g \star x_1$.

## Known-Order Effective Group Action (KO-EGA)

A KO-EGA is a tuple $(G, E, \star, E_0)$ where, in addition to the action above:

1. $G$ is finite, its decomposition $G \cong \bigoplus_i \mathbb{Z}/m_i\mathbb{Z}$ is known, and so is a minimal generating set $\langle g_i \rangle$.
2. $E$ is finite, with PPT algorithms for membership testing and unique bit-string representation.
3. There is a *distinguished* element $E_0 \in E$ whose representation is publicly known.
4. There is a PPT algorithm to evaluate $n g_i \star E$ for any natural number $n$ in range, $E \in E$, and generator $g_i$.

Knowing the order of $G$ lets you sample uniformly from $G$ (and from any
subgroup), which is what makes the model useful for designing protocols. The
two known post-quantum instantiations of KO-EGA are
[CSI-FiSh](https://eprint.iacr.org/2019/498) and
[SCALLOP](https://eprint.iacr.org/2023/058); both are below the NIST-1
quantum security level, so any scheme that builds on KO-EGA today should be
read as a feasibility result rather than a deployable construction.

## Hard problems

Once you have a regular group action, the standard discrete-log-style
hardness assumptions transfer almost verbatim:

**Group-Action Inverse Problem (GAIP).** Given a uniformly random $E \in E$,
find $g \in G$ such that $g \star E_0 = E$. This is the analog of discrete
log.

**Decisional Diffie–Hellman (DDH) for group actions.** Given
$(s_1 \star E_0,\; s_2 \star E_0,\; T)$ where either $T = (s_1 + s_2) \star E_0$
or $T$ is uniform in $E$, distinguish the two cases. This is the analog of the
classical DDH assumption.

The malleable commitment scheme described in the
[Malleable Commitments paper](../papers/malleable-commitments.md) is *hiding*
under DDH and *perfectly binding* by regularity of the action.
