# Model Loading and Parsing

This page covers the internals of `parse_nodes()` and how ONNX node types are mapped to the `Operator` enum.

---

## ONNX Parsing via Tract

The ONNX protobuf is parsed by the [`tract`](https://github.com/sonos/tract) library via `load_onnx_using_tract()`. Tract handles:

- Protocol buffer deserialization
- Operator attribute parsing (axis, perm, dilations, etc.)
- Constant folding for initializer tensors

The output of Tract is an untyped node graph that `parse_nodes()` walks to produce the typed `ComputationGraph`.

---

## ONNX Operator → Operator Enum Mapping

`parse_nodes()` iterates the Tract graph and maps each node type to an `Operator` variant. The mapping is implemented in `atlas-onnx-tracer/src/node/mod.rs` and the operator-specific files in `atlas-onnx-tracer/src/ops/`.

Key mappings:

| ONNX Op | `Operator` variant | Notes |
|---------|-------------------|-------|
| `Add`, `Sub`, `Mul` | `Add`, `Sub`, `Mul` | Element-wise |
| `Div` | `Div` | Integer Euclidean division |
| `Relu` | `ReLU` | |
| `Tanh`, `Erf`, `Sigmoid` | `Tanh`, `Erf`, `Sigmoid` | Neural Teleport activations |
| `Sin`, `Cos` | `Sin`, `Cos` | |
| `Einsum` | `Einsum(subscript)` | Subscript string (e.g. `"mk,kn->mn"`) stored |
| `Gemm`, `MatMul` | `Einsum("mk,kn->mn")` | Normalized to Einsum |
| `Softmax` | `Softmax(axis)` | Axis stored |
| `Gather` | `Gather` | Embedding lookup |
| `Reshape` | `Reshape` | |
| `Transpose` | `MoveAxis(perm)` | Permutation stored |
| `Slice` | `Slice` | |
| `Concat` | `Concat(axis)` | |
| `ReduceSum`, `ReduceMean`, `ReduceMax` | `ReduceSum`, `ReduceMean`, `ReduceMax` | |
| Initializer tensors | `Constant(tensor)` | Weights |
| Graph inputs | `Input(tensor)` | Populated with dummy data at load time |

---

## Identifying Graph Boundaries

`collect_input_nodes()` identifies which nodes correspond to the ONNX model's external inputs by matching node names against the ONNX `model.graph.input` list.

`collect_outputs()` similarly identifies output nodes by matching against `model.graph.output`.

These node indices are stored in `ComputationGraph.inputs` and `ComputationGraph.outputs` respectively, and are used by `Trace::io()` to extract `ModelExecutionIO` during the forward pass.

---

## Preserving Original Dimensions

After `.pad()`, every `output_dims` is a power-of-two. But the prover needs to know the *actual* output shape to correctly present the public output to the verifier.

`original_input_dims` and `original_output_dims` are populated before `.pad()` runs and persist through `build()`. When `Trace::io()` constructs `ModelExecutionIO`, it slices the padded output tensor back to the original shape using these stored dimensions.

---

## Example: GPT-2 Graph

A GPT-2 graph has ~300 nodes after parsing and pruning. The dominant operator types are:

- `Einsum`: attention projections (Q, K, V, O) and feedforward layers
- `Add`: residual connections and bias addition
- `Softmax`: attention score normalization
- `Gather`: token and position embedding lookups
- `ReLU` / `Tanh` / `Erf`: activation functions (model-dependent)
- `Reshape` / `MoveAxis`: head splitting and reshaping

The `inspect_ops.rs` example in `atlas-onnx-tracer/examples/` prints the full operator list for any model.
