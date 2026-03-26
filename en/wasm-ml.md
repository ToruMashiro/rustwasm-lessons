---
title: "Wasm + Machine Learning"
slug: wasm-ml
difficulty: advanced
tags: [api]
order: 41
description: Run machine learning inference in the browser using Wasm — build matrix operations, implement a neural network forward pass, and learn how Rust Wasm competes with TensorFlow.js.
starter_code: |
  use std::fmt;

  // A minimal neural network in plain Rust — no external crates.
  // Demonstrates the exact math that runs inside Wasm ML runtimes
  // like tract, onnxruntime, and candle.

  // --- Matrix type ---
  #[derive(Clone)]
  struct Matrix {
      rows: usize,
      cols: usize,
      data: Vec<f64>,
  }

  impl Matrix {
      fn new(rows: usize, cols: usize, data: Vec<f64>) -> Self {
          assert_eq!(data.len(), rows * cols, "Data size mismatch");
          Matrix { rows, cols, data }
      }

      fn zeros(rows: usize, cols: usize) -> Self {
          Matrix { rows, cols, data: vec![0.0; rows * cols] }
      }

      fn get(&self, r: usize, c: usize) -> f64 {
          self.data[r * self.cols + c]
      }

      fn set(&mut self, r: usize, c: usize, val: f64) {
          self.data[r * self.cols + c] = val;
      }

      // Matrix multiplication: (m x n) * (n x p) = (m x p)
      fn matmul(&self, other: &Matrix) -> Matrix {
          assert_eq!(self.cols, other.rows, "Dimension mismatch for matmul");
          let mut result = Matrix::zeros(self.rows, other.cols);
          for i in 0..self.rows {
              for j in 0..other.cols {
                  let mut sum = 0.0;
                  for k in 0..self.cols {
                      sum += self.get(i, k) * other.get(k, j);
                  }
                  result.set(i, j, sum);
              }
          }
          result
      }

      // Element-wise addition
      fn add(&self, other: &Matrix) -> Matrix {
          assert_eq!(self.rows, other.rows);
          assert_eq!(self.cols, other.cols);
          let data: Vec<f64> = self.data.iter()
              .zip(other.data.iter())
              .map(|(a, b)| a + b)
              .collect();
          Matrix::new(self.rows, self.cols, data)
      }

      // Broadcast add: add a 1xN bias to each row of an MxN matrix
      fn add_bias(&self, bias: &Matrix) -> Matrix {
          assert_eq!(bias.rows, 1);
          assert_eq!(self.cols, bias.cols);
          let mut result = self.clone();
          for r in 0..self.rows {
              for c in 0..self.cols {
                  result.data[r * self.cols + c] += bias.get(0, c);
              }
          }
          result
      }

      // Apply function element-wise
      fn map(&self, f: fn(f64) -> f64) -> Matrix {
          let data: Vec<f64> = self.data.iter().map(|&x| f(x)).collect();
          Matrix::new(self.rows, self.cols, data)
      }

      fn argmax_per_row(&self) -> Vec<usize> {
          (0..self.rows).map(|r| {
              let row_start = r * self.cols;
              let row = &self.data[row_start..row_start + self.cols];
              row.iter()
                  .enumerate()
                  .max_by(|a, b| a.1.partial_cmp(b.1).unwrap())
                  .map(|(i, _)| i)
                  .unwrap()
          }).collect()
      }
  }

  impl fmt::Display for Matrix {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          for r in 0..self.rows {
              write!(f, "  [")?;
              for c in 0..self.cols {
                  if c > 0 { write!(f, ", ")?; }
                  write!(f, "{:>8.4}", self.get(r, c))?;
              }
              writeln!(f, "]")?;
          }
          Ok(())
      }
  }

  // --- Activation functions ---

  fn relu(x: f64) -> f64 {
      if x > 0.0 { x } else { 0.0 }
  }

  fn softmax(m: &Matrix) -> Matrix {
      let mut result = Matrix::zeros(m.rows, m.cols);
      for r in 0..m.rows {
          let row_start = r * m.cols;
          let row = &m.data[row_start..row_start + m.cols];
          let max_val = row.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
          let exps: Vec<f64> = row.iter().map(|&x| (x - max_val).exp()).collect();
          let sum: f64 = exps.iter().sum();
          for c in 0..m.cols {
              result.set(r, c, exps[c] / sum);
          }
      }
      result
  }

  // --- Neural network layer ---

  struct DenseLayer {
      weights: Matrix,  // (input_size x output_size)
      bias: Matrix,     // (1 x output_size)
      activation: Option<fn(f64) -> f64>,
      name: String,
  }

  impl DenseLayer {
      fn forward(&self, input: &Matrix) -> Matrix {
          let z = input.matmul(&self.weights).add_bias(&self.bias);
          match self.activation {
              Some(f) => z.map(f),
              None => z,
          }
      }
  }

  // --- Simple neural network ---

  struct NeuralNetwork {
      layers: Vec<DenseLayer>,
  }

  impl NeuralNetwork {
      fn forward(&self, input: &Matrix) -> Matrix {
          let mut current = input.clone();
          for layer in &self.layers {
              println!("  Layer '{}': ({} x {}) -> ({} x {})",
                  layer.name,
                  current.rows, current.cols,
                  current.rows, layer.weights.cols);
              current = layer.forward(&current);
          }
          current
      }
  }

  fn main() {
      println!("=== Wasm ML: Neural Network Forward Pass ===\n");

      // Build a small network: 4 inputs -> 8 hidden -> 3 outputs
      // (e.g., classifying iris flowers: 4 features -> 3 species)

      let network = NeuralNetwork {
          layers: vec![
              DenseLayer {
                  weights: Matrix::new(4, 8, vec![
                      0.1, -0.2,  0.3,  0.4, -0.1,  0.2,  0.5, -0.3,
                     -0.4,  0.5, -0.1,  0.2,  0.3, -0.4,  0.1,  0.6,
                      0.2, -0.3,  0.4, -0.5,  0.1,  0.3, -0.2,  0.4,
                     -0.1,  0.4, -0.2,  0.3, -0.5,  0.1,  0.3, -0.1,
                  ]),
                  bias: Matrix::new(1, 8, vec![0.1, -0.1, 0.2, 0.0, 0.1, -0.2, 0.0, 0.1]),
                  activation: Some(relu),
                  name: "hidden1".to_string(),
              },
              DenseLayer {
                  weights: Matrix::new(8, 3, vec![
                      0.3, -0.2,  0.1,
                     -0.4,  0.5,  0.2,
                      0.1, -0.3,  0.4,
                      0.2,  0.1, -0.5,
                     -0.1,  0.3,  0.2,
                      0.4, -0.1,  0.3,
                     -0.2,  0.4, -0.1,
                      0.3, -0.2,  0.5,
                  ]),
                  bias: Matrix::new(1, 3, vec![0.1, -0.1, 0.05]),
                  activation: None,
                  name: "output".to_string(),
              },
          ],
      };

      // Two sample inputs (batch of 2)
      let input = Matrix::new(2, 4, vec![
          5.1, 3.5, 1.4, 0.2,   // sample 1 (Iris setosa features)
          6.7, 3.0, 5.2, 2.3,   // sample 2 (Iris virginica features)
      ]);

      println!("Input (2 samples, 4 features):");
      print!("{}", input);

      println!("\nForward pass:");
      let logits = network.forward(&input);

      println!("\nLogits (raw output):");
      print!("{}", logits);

      let probabilities = softmax(&logits);
      println!("Probabilities (after softmax):");
      print!("{}", probabilities);

      let predictions = probabilities.argmax_per_row();
      let classes = ["setosa", "versicolor", "virginica"];
      println!("Predictions:");
      for (i, &pred) in predictions.iter().enumerate() {
          println!("  Sample {}: class {} ({}) confidence {:.1}%",
              i + 1, pred, classes[pred],
              probabilities.get(i, pred) * 100.0);
      }
  }
expected_output: |
  === Wasm ML: Neural Network Forward Pass ===

  Input (2 samples, 4 features):
    [  5.1000,   3.5000,   1.4000,   0.2000]
    [  6.7000,   3.0000,   5.2000,   2.3000]

  Forward pass:
    Layer 'hidden1': (2 x 4) -> (2 x 8)
    Layer 'output': (2 x 8) -> (2 x 3)

  Logits (raw output):
    [  0.3530,   0.7190,   0.2910]
    [  1.3950,  -0.2990,   1.8740]
  Probabilities (after softmax):
    [  0.2957,   0.4264,   0.2779]
    [  0.3574,   0.0657,   0.5770]
  Predictions:
    Sample 1: class 1 (versicolor) confidence 42.6%
    Sample 2: class 2 (virginica) confidence 57.7%
---

## Why Run ML in the Browser with Wasm?

Machine learning inference in the browser has unique advantages:

```
  Traditional ML Pipeline           Wasm ML Pipeline

  ┌────────┐     ┌────────┐        ┌────────────────────┐
  │ Client │────>│ Server │        │ Browser             │
  │        │     │ (GPU)  │        │                     │
  │ Send   │     │ Run    │        │ ┌────────────────┐  │
  │ data   │     │ model  │        │ │ Wasm ML Runtime│  │
  │ over   │     │ return │        │ │                │  │
  │ network│<────│ result │        │ │ Load model     │  │
  └────────┘     └────────┘        │ │ Run inference  │  │
                                   │ │ Return result  │  │
  ● Network latency                │ └────────────────┘  │
  ● Privacy: data leaves device    │                     │
  ● Server cost (GPU $$$)          │ ● Zero latency     │
                                   │ ● Data stays local  │
                                   │ ● No server cost    │
                                   └────────────────────┘
```

| Factor              | Server ML          | Wasm ML (Browser)       | WebGPU ML            |
|---------------------|--------------------|-------------------------|----------------------|
| Latency             | Network RTT        | Instant (local)         | Instant (local)      |
| Privacy             | Data sent to server| Data stays on device    | Data stays on device |
| Hardware            | Server GPU         | CPU (all threads)       | Client GPU           |
| Cost                | Per-request        | Free (client CPU)       | Free (client GPU)    |
| Model size limit    | Unlimited          | ~100 MB practical       | ~1 GB practical      |
| Accuracy            | Full precision     | Full precision          | May use f16          |
| Offline capable     | No                 | Yes (Service Worker)    | Yes                  |
| Browser support     | N/A (server)       | Universal               | Chrome, Edge, Firefox|

## ML Runtimes for Wasm

### tract (ONNX Runtime in Rust)

`tract` is the most mature Rust ML runtime that compiles to Wasm:

```rust
use tract_onnx::prelude::*;

// Load an ONNX model
let model = tract_onnx::onnx()
    .model_for_path("model.onnx")?
    .with_input_fact(0, f32::fact([1, 3, 224, 224]).into())?
    .into_optimized()?
    .into_runnable()?;

// Run inference
let input: Tensor = tract_ndarray::Array4::from_shape_fn(
    (1, 3, 224, 224),
    |(_, c, y, x)| pixel_value(image, c, y, x)
).into();

let result = model.run(tvec!(input.into()))?;
let output = result[0].to_array_view::<f32>()?;
```

### candle (Hugging Face)

```rust
use candle_core::{Tensor, Device};

let device = Device::Cpu;  // Wasm always uses CPU
let weights = Tensor::from_vec(vec![1.0f32, 2.0, 3.0], (3,), &device)?;
let input = Tensor::from_vec(vec![0.5f32, 0.3, 0.2], (3,), &device)?;
let output = (weights * input)?;
```

### Comparison of Wasm ML runtimes

```
  ┌─────────────┬──────────────┬───────────────┬────────────────┐
  │ Runtime     │ Model Format │ Wasm Support  │ Best For       │
  ├─────────────┼──────────────┼───────────────┼────────────────┤
  │ tract       │ ONNX, TF     │ Excellent     │ Classic models │
  │ candle      │ safetensors  │ Good          │ LLMs, HF models│
  │ burn        │ burn format  │ Experimental  │ Custom training│
  │ ort (onnx)  │ ONNX         │ Via wasm-pack │ ONNX ecosystem │
  └─────────────┴──────────────┴───────────────┴────────────────┘
```

## Tensor Operations in Detail

The core of any ML runtime is tensor math. Here are the key operations:

### Matrix Multiplication (GEMM)

The most performance-critical operation in neural networks:

```
  Matrix A (2x3)     Matrix B (3x2)     Result (2x2)

  ┌─────────────┐   ┌─────────┐       ┌───────────┐
  │ 1   2   3   │   │ 7   8   │       │ 58    64  │
  │             │ × │ 9  10   │  =    │           │
  │ 4   5   6   │   │11  12   │       │139   154  │
  └─────────────┘   └─────────┘       └───────────┘

  result[0][0] = 1*7 + 2*9  + 3*11  = 58
  result[0][1] = 1*8 + 2*10 + 3*12  = 64
  result[1][0] = 4*7 + 5*9  + 6*11  = 139
  result[1][1] = 4*8 + 5*10 + 6*12  = 154
```

### Activation Functions

```
  ReLU                    Sigmoid                  Softmax
  f(x) = max(0, x)       f(x) = 1/(1+e^-x)       f(xi) = e^xi / sum(e^xj)

  output │     ╱          output │    ──────       output │   ╱
         │    ╱                  │  ╱              (prob) │  ╱
         │   ╱              0.5 │╱                       │╱────
  ───────┼──╱            ───────┼──────            ──────┼──────
         │               ╱     │                        │
         │             ╱       │                        │
  ───────┘                                              │
```

| Activation | Formula              | Range      | Used For                  |
|------------|----------------------|------------|---------------------------|
| ReLU       | max(0, x)            | [0, inf)   | Hidden layers (default)   |
| Sigmoid    | 1 / (1 + e^-x)      | (0, 1)     | Binary classification     |
| Softmax    | e^xi / sum(e^xj)    | (0, 1)     | Multi-class output        |
| Tanh       | (e^x - e^-x)/(e^x + e^-x) | (-1, 1) | RNNs, hidden layers  |

## Loading Models in the Browser

```
  ┌─────────────────────────────────────────────────────┐
  │  Model Loading Pipeline                             │
  │                                                     │
  │  1. fetch("model.onnx")                             │
  │     │                                               │
  │     ▼                                               │
  │  2. ArrayBuffer (raw bytes)                         │
  │     │                                               │
  │     ▼                                               │
  │  3. Pass to Wasm: load_model(&bytes)                │
  │     │                                               │
  │     ├── Parse model graph (protobuf/flatbuf)        │
  │     ├── Allocate weight tensors in Wasm memory      │
  │     ├── Optimize graph (fuse ops, constant fold)    │
  │     └── Return opaque handle                        │
  │     │                                               │
  │     ▼                                               │
  │  4. Ready for inference: predict(handle, input)     │
  └─────────────────────────────────────────────────────┘
```

```javascript
// JavaScript side
import init, { WasmModel } from './pkg/ml_wasm';

async function loadAndPredict() {
    await init();

    // Fetch model bytes
    const response = await fetch('/models/mobilenet_v2.onnx');
    const modelBytes = new Uint8Array(await response.arrayBuffer());

    // Load into Wasm
    const model = WasmModel.load(modelBytes);

    // Prepare input (224x224 RGB image as flat Float32Array)
    const input = preprocessImage(imageElement);

    // Run inference
    const output = model.predict(input);
    const topClass = argmax(output);
    console.log(`Prediction: ${IMAGENET_CLASSES[topClass]}`);
}
```

## Performance: Wasm vs TensorFlow.js

Benchmarks on a typical laptop (MobileNet V2, single image):

```
  Inference Time (ms) — lower is better

  TF.js (WebGL)  ████████████  12ms
  Wasm (tract)   ██████████████████  18ms
  TF.js (Wasm)   ████████████████████  20ms
  TF.js (CPU/JS) █████████████████████████████████████████  42ms

  ← faster                                     slower →
```

| Runtime              | MobileNet V2 | ResNet50   | BERT (base) |
|----------------------|-------------|------------|-------------|
| TF.js WebGL          | ~12ms       | ~45ms      | ~120ms      |
| Wasm (tract)         | ~18ms       | ~80ms      | ~300ms      |
| TF.js Wasm backend   | ~20ms       | ~90ms      | ~350ms      |
| TF.js CPU (pure JS)  | ~42ms       | ~200ms     | ~800ms      |
| Native (PyTorch CPU)  | ~8ms        | ~30ms      | ~80ms       |

**Key takeaways:**
- Wasm is **2-3x faster** than pure JavaScript
- WebGL/WebGPU beats Wasm for large models (GPU parallelism)
- Wasm wins for **small models** where GPU setup overhead dominates
- Wasm is more **predictable** (no GPU driver variability)

## When to Use Wasm ML vs WebGPU

```
  Decision Matrix:

                     Small model           Large model
                     (< 10M params)        (> 100M params)
  ┌─────────────────┬────────────────────┬────────────────────┐
  │ Need offline    │  Wasm ✓            │  Wasm (if fits)    │
  │ support?        │                    │  or WebGPU         │
  ├─────────────────┼────────────────────┼────────────────────┤
  │ Consistent      │  Wasm ✓            │  Wasm              │
  │ performance?    │  (no GPU variance) │  (slower but steady)│
  ├─────────────────┼────────────────────┼────────────────────┤
  │ Maximum speed?  │  Wasm ✓            │  WebGPU ✓          │
  │                 │  (GPU overhead     │  (GPU parallelism  │
  │                 │   not worth it)    │   dominates)       │
  ├─────────────────┼────────────────────┼────────────────────┤
  │ Browser compat? │  Wasm ✓            │  Wasm ✓            │
  │                 │  (universal)       │  WebGPU still new  │
  └─────────────────┴────────────────────┴────────────────────┘
```

## Practical Use Cases

| Application                    | Model Type      | Why Wasm?                         |
|-------------------------------|-----------------|-----------------------------------|
| Spam detection                | Logistic reg.   | Tiny model, instant prediction    |
| Image classification          | MobileNet       | Runs on any device, offline       |
| Text autocomplete             | Small RNN       | Low latency, privacy              |
| Pose estimation (camera)      | PoseNet         | Real-time, no server round-trip   |
| Document OCR                  | CRNN            | Sensitive documents stay local    |
| Audio keyword detection       | Small CNN       | Always-on, low power              |
| Anomaly detection (IoT)       | Autoencoder     | Edge device, no connectivity      |

## Summary

Wasm brings ML inference to the browser with full precision, offline capability, and zero server cost. Use `tract` for ONNX models, `candle` for Hugging Face models, or build custom tensor operations in Rust. Wasm ML is 2-3x faster than pure JavaScript and is the best choice for small-to-medium models where GPU overhead is not worth it. For large models (LLMs, diffusion), combine Wasm with WebGPU for GPU acceleration.
