# Phase 1: ONNX to ComputationGraph

The first phase transforms an ONNX file into an in-memory typed node graph (`ComputationGraph`) with integer arithmetic semantics. This is entirely handled by `atlas-onnx-tracer`.

---

## Entry Point

```rust
let model = Model::load("path/to/model.onnx", &Default::default())?;
```

`Model::load()` (`atlas-onnx-tracer/src/model/mod.rs`) calls `Model::load_onnx_model()` (`atlas-onnx-tracer/src/model/load.rs`), which runs a builder chain:

```rust
ModelLoader::new()
    .load_onnx_using_tract()   // Tract parses the .onnx protobuf
    .parse_nodes()             // Each ONNX node → ComputationNode
    .collect_input_nodes()     // Identify graph input nodes
    .collect_outputs()         // Identify graph output nodes
    .prune()                   // Remove unreachable nodes
    .pad()                     // Pad dims to next power-of-two
    .build()                   // Assemble ComputationGraph
```

---

## ComputationGraph

The result is a `ComputationGraph`:

```rust
pub struct ComputationGraph {
    pub nodes: BTreeMap<usize, ComputationNode>,
    pub inputs: Vec<usize>,
    pub outputs: Vec<usize>,
    pub original_input_dims: HashMap<usize, Vec<usize>>,
    pub original_output_dims: HashMap<usize, Vec<usize>>,
}
```

The `nodes` BTreeMap is keyed by a monotonically increasing integer index. BTreeMap insertion order is approximately topological (parents before children), matching the order ONNX nodes are declared in the protobuf.

---

## ComputationNode

```rust
pub struct ComputationNode {
    pub idx: usize,
    pub operator: Operator,
    pub inputs: Vec<usize>,      // ← DAG edges: indices of input nodes
    pub output_dims: Vec<usize>, // ← shape after padding
}
```

The `inputs` field is the edge list of the computation DAG. If `node.inputs = [3, 7]`, then node 3 and node 7 are the left and right operands of this node.

---

## Operator Parsing

`parse_nodes()` maps each ONNX node type to an `Operator` enum variant. The ~35 variants cover all supported operations:

```rust
pub enum Operator {
    Add, Sub, Mul, Div, Neg, Square, Cube,
    ReLU, Tanh, Sigmoid, Erf, Sin, Cos,
    Einsum(String), Softmax(usize),
    Gather, Reshape, Concat, Slice, MoveAxis,
    ReduceSum, ReduceMean, ReduceMax,
    Input(Tensor<i32>), Constant(Tensor<i32>),
    Identity,
    // ...
}
```

`Input` and `Constant` nodes carry their tensor data directly in the enum variant. Weight tensors from the ONNX initializer list are parsed as `Constant` nodes.

---

## Power-of-Two Padding

The `.pad()` step rounds up every tensor dimension to the next power-of-two. This is done at load time, before any proof is generated.

**Why?** Every tensor will eventually be encoded as a multilinear polynomial (MLE) over a Boolean hypercube $\{0,1\}^k$. An MLE over $\{0,1\}^k$ has exactly $2^k$ evaluations. Padding ensures this holds for every tensor without requiring special handling at proof time.

The `original_input_dims` and `original_output_dims` fields preserve the un-padded shapes so that the prover can correctly slice the model's true inputs and outputs from the padded tensors when constructing `ModelExecutionIO`.

---

## Dead-Node Elimination

The `.prune()` step performs a reachability analysis from the output nodes backwards through the `inputs` edge lists, removing any node not on a path from an input to an output. This reduces proof size by eliminating unused operations.

---

## Next

After `Model::load()`, the `ComputationGraph` is ready for the forward pass. See [Phase 2: Forward Pass and Trace](./trace.md).

For details on how individual ONNX node types are mapped to `Operator` variants, see [Model Loading and Parsing](./model-loading.md).

For the quantization scheme applied to weight tensors, see [Quantization](./quantization.md).
