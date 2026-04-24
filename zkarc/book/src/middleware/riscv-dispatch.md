# RISC-V dispatch program

> **Scope.** This is an optional hardening step for specific, standardised
> tool calls (e.g., a fixed-format HTTP POST), not a general solution.
> Tool calls in practice are arbitrary (HTTP APIs, database queries, shell
> commands, smart contract calls, file operations) and user-dependent.
> Wrapping all of them in a RISC-V guest is not practical. For the general
> case, PreFlight's role is to **prove the policy check** and **bind the
> proof to the action text**; dispatch is left to the user's agent
> framework. See the [comparison page](./comparison.md) for how the proof
> layer integrates with any framework.

For the narrow case where the tool call format is simple and standardised,
a RISC-V guest program provides the strongest form of execution binding:
it verifies the PreFlight proof and commits to a dispatch instruction, all
inside the zkVM. The proof attests that the dispatch is a deterministic
consequence of a verified policy check.

## The problem the guest solves

The [comparison page](./comparison.md) establishes that PreFlight is a proof
layer that plugs into existing framework hooks. But the hook still runs on
the operator's machine. The operator could:

- Call PreFlight, get SAT, and then dispatch a *different* action.
- Skip the PreFlight call entirely and dispatch without a proof.
- Modify the action between the proof check and the dispatch.

A signed log or an API response does not bind the *verified action* to the
*dispatched action*. The RISC-V dispatch program closes this gap.

## Design

The guest program runs inside Jolt's zkVM. It has no network access, no
filesystem, no syscalls. It is pure computation: inputs in, outputs out.

### Inputs (private witness + public inputs)

| Input | Visibility | What it is |
|-------|-----------|------------|
| `proof` | Private witness | The PreFlight proof $\pi$ (output of `/v1/checkIt`) |
| `action` | Public input | The action text that was checked |
| `policy_commitment` | Public input | The committed policy $\bar{P}$ |
| `endpoint` | Public input | The target endpoint / tool call destination |

### What the guest does

```rust
// Guest program (runs inside Jolt zkVM)
// No network, no I/O, no syscalls.

fn main(
    proof: &[u8],           // private witness
    action: &str,           // public input
    policy_commitment: &[u8], // public input
    endpoint: &str,         // public input
) -> DispatchInstruction {
    // 1. Deserialise and verify the PreFlight proof
    let verdict = jolt_verify(proof, action, policy_commitment);

    // 2. Assert the verdict is SAT
    assert!(verdict == Verdict::SAT);

    // 3. Output a committed dispatch instruction
    DispatchInstruction {
        endpoint: endpoint.to_string(),
        action: action.to_string(),
        verified: true,
    }
}
```

### Output (public)

A `DispatchInstruction`: the endpoint, the action text, and a `verified`
flag. This is the committed output of the proved computation.

### What happens outside the zkVM

A host-side wrapper reads the `DispatchInstruction` from the guest's output
and performs the actual HTTP call / API invocation / tool execution. The
host-side dispatch is deterministic given the output: if `verified == true`,
dispatch `action` to `endpoint`.

```
┌──────────────────────────────────────────┐
│              Jolt zkVM                   │
│                                          │
│  proof (witness) ──► Verify proof        │
│  action (public) ──► Assert SAT          │
│  endpoint (public)──► Build instruction  │
│                          │               │
│                          ▼               │
│              DispatchInstruction          │
│              (endpoint, action, verified)│
└──────────────┬───────────────────────────┘
               │
               │  π_dispatch (Jolt proof of guest execution)
               │
               ▼
┌──────────────────────────────────────────┐
│           Host-side wrapper              │
│                                          │
│  Read DispatchInstruction                │
│  If verified: HTTP POST to endpoint      │
│  Attach π_dispatch to the request        │
│                                          │
└──────────────┬───────────────────────────┘
               │
               ▼
         Counterparty
         (verifies π_dispatch)
```

## What the proof attests

The Jolt proof $\pi_{\text{dispatch}}$ attests:

> "A program that verifies a PreFlight proof and outputs a dispatch
> instruction was executed. The PreFlight proof verified successfully
> (SAT) for this action text against this policy commitment. The dispatch
> instruction targets this endpoint with this action."

The counterparty verifies $\pi_{\text{dispatch}}$ and knows:

1. A policy check was performed (inner proof verified inside the guest).
2. The check passed (SAT).
3. The dispatch instruction matches the action that arrived.
4. No one could have tampered with the action between verification and
   dispatch, because the dispatch instruction is a deterministic output of
   proved code.

## What remains outside the proof

The host-side wrapper makes the actual network call. This is outside the
zkVM and therefore outside the proof. The remaining trust assumptions:

- **The host ran the wrapper faithfully.** The host could read the
  `DispatchInstruction` and not dispatch, or dispatch to a different
  endpoint. But it cannot forge $\pi_{\text{dispatch}}$: the counterparty
  checks the proof and compares the proved endpoint/action against what
  it received. A mismatch is detectable.
- **Network delivery.** The action could fail in transit (network error,
  timeout). This is an availability problem, not an integrity problem.

The integrity chain is complete: proved policy check → proved dispatch
instruction → counterparty verifies both match what it received.

## Why RISC-V and not native code

The guest program must run inside a zkVM to produce a proof. Jolt's zkVM
targets RISC-V. The guest is compiled to RISC-V and its binary is committed
(the binary commitment is part of the Jolt proof structure). A different
binary produces a different commitment, which the counterparty detects.

The guest is small: deserialise a proof, run a verifier, output a struct.
No ML, no solver, no heavy computation. The Jolt proving overhead for this
guest should be sub-second.

## Recursive proof structure

$\pi_{\text{dispatch}}$ is a Jolt proof that *contains* a verified
PreFlight proof. This is recursive verification: the inner proof (PreFlight
/ policy check) is verified inside the outer proof (dispatch program).

The counterparty only needs to verify the outer proof. It does not need
access to the inner proof, the policy, or the solver. Everything is
absorbed into $\pi_{\text{dispatch}}$.

This is exactly the Tier 2 (recursive verification) pattern from the
[proof aggregation](../paper/proof-aggregation.md) design, applied to
execution binding rather than proof composition.

## Implementation notes

- The Jolt verifier needs to be callable from the guest program. This
  requires the verifier to compile to RISC-V. The verifier is already
  Rust; RISC-V compilation should be feasible but needs testing.
- The `DispatchInstruction` struct must be serialisable as a Jolt public
  output. Jolt outputs are field elements; the action text and endpoint
  would be hashed and the hash committed as the public output, with the
  plaintext provided alongside for the counterparty to check.
- The guest binary must be reproducibly built so the binary commitment is
  auditable. See the [trust-your-own-setup](./middleware.md#trust-your-own-setup)
  discussion.
