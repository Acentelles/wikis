# Gather and Embeddings

The `Gather` operator implements embedding table lookups, the fundamental operation in the first layer of language models that maps token IDs to embedding vectors.

---

## What Gather Does

Given a weight table $W \in \mathbb{Z}^{V \times d}$ (vocabulary size $V$, embedding dimension $d$) and an index tensor $\text{idx} \in \mathbb{Z}^{N}$ (token IDs), Gather computes:

$$\text{out}[t, :] = W[\text{idx}[t], :]$$

Each row $\text{idx}[t]$ selects a row of the weight table.

---

## Why Gather is Different from Activation Lookups

Activation lookups (ReLU, Tanh, etc.) use **one-hot** address polynomials: for each element, exactly one table entry is selected, and the table is a small fixed function.

Gather uses a **dense** address polynomial: each query selects an entire *row* of a large weight table. The address values are arbitrary token IDs, and there is no one-hot structure.

---

## Committed Polynomial: GatherRa

`GatherRa(node_idx)` is a **dense** MLE encoding the lookup addresses:

$$\text{GatherRa}[t] = \text{idx}[t] / V$$

normalized to $[0, 1)$ as a field element. This polynomial is committed as a standard dense MLE, not a one-hot polynomial.

The witness generation reads the `idx` tensor from the trace and converts each index to a field element:

```rust
let addresses = idx.iter().map(|&i| F::from(i as u64)).collect::<Vec<_>>();
MultilinearPolynomial::from(addresses)
```

---

## The Gather Sumcheck

The constraint is:

$$\text{out}(x) = W(\text{GatherRa}(x))$$

where $W$ is the weight table MLE.

In sumcheck form, a Shout/read-raf protocol proves that each output row was correctly read from the weight table at the index given by `GatherRa`. The sumcheck operates over the cycle dimension (one cycle per token position) rather than the table dimension.

The proof type is `ProofType::Execution` for the main read sumcheck.

---

## Weight Table as a Committed Polynomial

The weight table $W$ is itself a committed polynomial; it is a `Constant` node in the computation graph, with its MLE committed at preprocessing time. The Gather proof links the output to the weight table commitment via the accumulator.

---

## Position Embeddings

In GPT-style models, both token embeddings and position embeddings are looked up via `Gather`. The two `Gather` outputs are then added together (via an `Add` node). The proof handles both embeddings independently as two separate Gather nodes.

---

## Difference from Neural Teleport

| Property | Gather | Neural Teleport |
|----------|--------|----------------|
| Address polynomial | Dense MLE | One-hot sparse |
| Table content | Model weights (arbitrary) | Fixed math function |
| Table size | Vocabulary × embedding dim | Small (bounded by τ) |
| Proof type | `Execution` | `NeuralTeleport` |
