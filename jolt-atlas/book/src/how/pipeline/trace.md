# Phase 2: Forward Pass and Trace

Before any proof is generated, Jolt Atlas runs the model in full to record every intermediate tensor. This recording is called the **Trace**.

---

## Entry Point

```rust
// Inside ONNXProof::prove()
let trace = pp.model().trace(inputs);
let io = Trace::io(&trace, pp.model());
```

`Model::trace(inputs)` calls `Model::execute_graph()` and wraps the result in a `Trace`.

---

## execute_graph

`Model::execute_graph()` in `atlas-onnx-tracer/src/model/execute.rs` is a simple forward pass:

```rust
for (idx, node) in graph.nodes.iter() {
    let input_tensors = node.inputs.iter()
        .map(|&i| outputs[i].clone())
        .collect::<Vec<_>>();
    let output = node.operator.f(&input_tensors);
    outputs.insert(*idx, output);
}
```

It iterates nodes in BTreeMap insertion order (which is approximately topological, since ONNX specifies a topological ordering). For each node it calls `operator.f()` with the already-computed outputs of upstream nodes, then stores the result.

All arithmetic is **integer (`i32`) arithmetic**: no floating point, no field elements, no ZK machinery. The operator implementations in `atlas-onnx-tracer/src/ops/` are pure `i32 → i32` functions.

---

## The Trace

```rust
pub struct Trace {
    pub node_outputs: BTreeMap<usize, Tensor<i32>>,
}
```

Every intermediate tensor at every node is stored. For GPT-2 (125M parameters), this means storing hundreds of large `i32` tensors; the trace is typically the largest in-memory data structure during proving.

---

## ModelExecutionIO

`Trace::io(&trace, model)` extracts the public inputs and outputs:

```rust
pub struct ModelExecutionIO {
    pub inputs: Vec<Tensor<i32>>,   // model's external inputs
    pub outputs: Vec<Tensor<i32>>,  // model's external outputs
}
```

`ModelExecutionIO` is passed to the verifier as the **public statement** being proved. The verifier does not see any intermediate activation values, only the input/output boundary.

---

## Independence from ZK

The tracer (`atlas-onnx-tracer`) has no dependency on `jolt-atlas-core` or `joltworks`. It is a standalone execution engine. This separation means:

- You can use the tracer independently for inference, quantization error analysis, or model inspection.
- The proof system can be tested against the tracer's outputs without requiring a full end-to-end build.
- The forward-pass code is simple and auditable without cryptographic complexity.

---

## Next

With the `Trace` in hand, `ONNXProof::prove()` proceeds to Phase 3: [witness commitment](./witness-commitment.md) and [the IOP loop](./iop-loop.md).
