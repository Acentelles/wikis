# Hidden-policy trust

## The B2B question

In hidden-policy mode, $P_{\mathsf{NL}}$ and $P_{\mathsf{SMT}}$ are private
witnesses committed as $\bar{P}$. The verifier sees the verdict and the
commitment but cannot inspect the policy. How does a counterparty trust a
proof against a policy it cannot see?

## Three levels of trust

1. **Auditor-mediated.** A trusted third party (auditor, regulator, industry
   body) inspects the policy, verifies it meets a standard, and signs the
   commitment $\bar{P}$. The counterparty trusts the auditor's signature.
   Closest to how compliance certification works today.

2. **Meta-property proofs.** The policy owner proves in ZK that the committed
   policy satisfies certain structural properties ("contains at least one rule
   about data retention," "forbids transactions above threshold X") without
   revealing the full rule set. Stronger than auditor trust, does not require
   a trusted third party, but requires defining meta-properties amenable to
   ZK proving.

3. **Reputation-backed commitment.** The counterparty trusts the principal's
   brand/reputation for the *content* of the policy, but gets cryptographic
   assurance that the policy was *executed correctly*. Weakest form but most
   immediately deployable: cleanly separates "is this a good policy?"
   (reputation) from "was this policy actually enforced?" (proof).

## Deployment path

Most early deployments will start with reputation-backed commitments and
upgrade to auditor-mediated or meta-property trust as the ecosystem matures.
