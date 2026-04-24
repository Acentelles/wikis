# Related work

## Heuristic guardrail frameworks

LangChain/LangGraph, NeMo Guardrails (Colang DSL), Guardrails AI (RAIL/Pydantic),
Llama Guard (LLM-based safety classifier), Semantic Integrity Constraints
(formal contracts without crypto binding).

All centralized: the guardrail runs on the operator's infrastructure, the
verifier must trust the operator. No cryptographic proof, no policy privacy,
non-deterministic.

## TEE-based attestation

Proof-of-Guardrail runs the guardrail inside a TEE, producing a hardware-signed
attestation. Partially addresses binding but requires trusting the TEE hardware
manufacturer. Not succinctly verifiable (attestation chain scales with workload
complexity). TEE hardware is expensive and limits deployment to specialised
infrastructure.

## ZK for neural networks

EZKL (Halo2-based, ONNX to ZK), zkLLM (tensor-lookup + ZK attention),
DeepProve (sumcheck + GKR, sublinear proving), NanoZK (layerwise constant-size
proofs). All prove a model executed correctly; none addresses guardrail
pipelines, policy formalisation, solver verification, or multi-proof
composition.

## Comparison table

| Property | LangChain / NeMo | Llama Guard | TEE | zkARc |
|----------|-----------------|-------------|-----|-------|
| Verification type | Heuristic | LLM classifier | Heuristic in TEE | Formal (SMT) |
| Trust model | Operator | Operator | TEE hardware | Trustless (ZKP) |
| Binding | None | None | TEE attestation | ZKP |
| Policy privacy | No | No | No | Yes |
| Prove-once-verify-many | No | No | Partial | Yes |
| Deterministic | No | No | No | Yes |
| Model specificity | No | No | Code hash | Committed model |
