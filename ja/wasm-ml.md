---
title: "Wasm + 機械学習"
slug: wasm-ml
difficulty: advanced
tags: [api]
order: 41
description: Wasmを使ってブラウザで機械学習推論を実行 — 行列演算の構築、ニューラルネットワークのフォワードパス実装、Rust WasmとTensorFlow.jsの比較を学びます。
starter_code: |
  use std::fmt;

  // 外部クレートなしのプレーンRustによるミニマルなニューラルネットワーク。
  // tract、onnxruntime、candleなどのWasm MLランタイム内部で
  // 実行されるのと同じ数学をデモします。

  // --- 行列型 ---
  #[derive(Clone)]
  struct Matrix {
      rows: usize,
      cols: usize,
      data: Vec<f64>,
  }

  impl Matrix {
      fn new(rows: usize, cols: usize, data: Vec<f64>) -> Self {
          assert_eq!(data.len(), rows * cols, "データサイズの不一致");
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

      // 行列乗算: (m x n) * (n x p) = (m x p)
      fn matmul(&self, other: &Matrix) -> Matrix {
          assert_eq!(self.cols, other.rows, "matmulの次元不一致");
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

      // 要素ごとの加算
      fn add(&self, other: &Matrix) -> Matrix {
          assert_eq!(self.rows, other.rows);
          assert_eq!(self.cols, other.cols);
          let data: Vec<f64> = self.data.iter()
              .zip(other.data.iter())
              .map(|(a, b)| a + b)
              .collect();
          Matrix::new(self.rows, self.cols, data)
      }

      // ブロードキャスト加算: MxN行列の各行に1xNバイアスを加算
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

      // 要素ごとに関数を適用
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

  // --- 活性化関数 ---

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

  // --- ニューラルネットワーク層 ---

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

  // --- シンプルなニューラルネットワーク ---

  struct NeuralNetwork {
      layers: Vec<DenseLayer>,
  }

  impl NeuralNetwork {
      fn forward(&self, input: &Matrix) -> Matrix {
          let mut current = input.clone();
          for layer in &self.layers {
              println!("  層 '{}': ({} x {}) -> ({} x {})",
                  layer.name,
                  current.rows, current.cols,
                  current.rows, layer.weights.cols);
              current = layer.forward(&current);
          }
          current
      }
  }

  fn main() {
      println!("=== Wasm ML: ニューラルネットワーク フォワードパス ===\n");

      // 小さなネットワークを構築: 4入力 -> 8隠れ -> 3出力
      // (例: アヤメの分類: 4特徴量 -> 3品種)

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

      // 2つのサンプル入力（バッチサイズ2）
      let input = Matrix::new(2, 4, vec![
          5.1, 3.5, 1.4, 0.2,   // サンプル1（アヤメsetosaの特徴量）
          6.7, 3.0, 5.2, 2.3,   // サンプル2（アヤメvirginicaの特徴量）
      ]);

      println!("入力（2サンプル、4特徴量）:");
      print!("{}", input);

      println!("\nフォワードパス:");
      let logits = network.forward(&input);

      println!("\nロジット（生の出力）:");
      print!("{}", logits);

      let probabilities = softmax(&logits);
      println!("確率（softmax後）:");
      print!("{}", probabilities);

      let predictions = probabilities.argmax_per_row();
      let classes = ["setosa", "versicolor", "virginica"];
      println!("予測結果:");
      for (i, &pred) in predictions.iter().enumerate() {
          println!("  サンプル {}: クラス {} ({}) 確信度 {:.1}%",
              i + 1, pred, classes[pred],
              probabilities.get(i, pred) * 100.0);
      }
  }
expected_output: |
  === Wasm ML: ニューラルネットワーク フォワードパス ===

  入力（2サンプル、4特徴量）:
    [  5.1000,   3.5000,   1.4000,   0.2000]
    [  6.7000,   3.0000,   5.2000,   2.3000]

  フォワードパス:
    層 'hidden1': (2 x 4) -> (2 x 8)
    層 'output': (2 x 8) -> (2 x 3)

  ロジット（生の出力）:
    [  0.3530,   0.7190,   0.2910]
    [  1.3950,  -0.2990,   1.8740]
  確率（softmax後）:
    [  0.2957,   0.4264,   0.2779]
    [  0.3574,   0.0657,   0.5770]
  予測結果:
    サンプル 1: クラス 1 (versicolor) 確信度 42.6%
    サンプル 2: クラス 2 (virginica) 確信度 57.7%
---

## なぜブラウザでWasmを使ってMLを実行するのか?

ブラウザでの機械学習推論には独自の利点があります：

```
  従来のMLパイプライン              Wasm MLパイプライン

  ┌────────┐     ┌────────┐        ┌────────────────────┐
  │ クライアント│────>│ サーバー │       │ ブラウザ             │
  │        │     │ (GPU)  │        │                     │
  │ データを│     │ モデルを│        │ ┌────────────────┐  │
  │ ネット  │     │ 実行   │        │ │ Wasm MLランタイム│  │
  │ ワーク  │     │ 結果を │        │ │                │  │
  │ 経由で  │     │ 返す   │        │ │ モデルをロード   │  │
  │ 送信   │<────│        │        │ │ 推論を実行      │  │
  └────────┘     └────────┘        │ │ 結果を返す      │  │
                                   │ └────────────────┘  │
  ● ネットワーク遅延               │                     │
  ● プライバシー: データが端末外へ   │ ● 遅延ゼロ          │
  ● サーバーコスト (GPU $$$)        │ ● データは端末に留まる │
                                   │ ● サーバーコストなし   │
                                   └────────────────────┘
```

| 要素                  | サーバーML           | Wasm ML（ブラウザ）         | WebGPU ML              |
|----------------------|--------------------|---------------------------|----------------------|
| レイテンシ             | ネットワークRTT      | 即時（ローカル）             | 即時（ローカル）        |
| プライバシー           | データがサーバーに送信 | データは端末に留まる          | データは端末に留まる    |
| ハードウェア           | サーバーGPU          | CPU（全スレッド）            | クライアントGPU        |
| コスト                | リクエストごと        | 無料（クライアントCPU）       | 無料（クライアントGPU） |
| モデルサイズ制限        | 無制限              | 実用的には約100 MB           | 実用的には約1 GB       |
| 精度                  | フル精度            | フル精度                    | f16を使う場合あり       |
| オフライン対応         | 不可                | 可能（Service Worker）       | 可能                  |
| ブラウザサポート        | N/A（サーバー）      | ユニバーサル                 | Chrome、Edge、Firefox  |

## Wasm用MLランタイム

### tract（RustのONNXランタイム）

`tract`はWasmにコンパイルできる最も成熟したRust MLランタイムです：

```rust
use tract_onnx::prelude::*;

// ONNXモデルをロード
let model = tract_onnx::onnx()
    .model_for_path("model.onnx")?
    .with_input_fact(0, f32::fact([1, 3, 224, 224]).into())?
    .into_optimized()?
    .into_runnable()?;

// 推論を実行
let input: Tensor = tract_ndarray::Array4::from_shape_fn(
    (1, 3, 224, 224),
    |(_, c, y, x)| pixel_value(image, c, y, x)
).into();

let result = model.run(tvec!(input.into()))?;
let output = result[0].to_array_view::<f32>()?;
```

### candle（Hugging Face）

```rust
use candle_core::{Tensor, Device};

let device = Device::Cpu;  // Wasmでは常にCPUを使用
let weights = Tensor::from_vec(vec![1.0f32, 2.0, 3.0], (3,), &device)?;
let input = Tensor::from_vec(vec![0.5f32, 0.3, 0.2], (3,), &device)?;
let output = (weights * input)?;
```

### Wasm MLランタイムの比較

```
  ┌─────────────┬──────────────┬───────────────┬────────────────┐
  │ ランタイム    │ モデル形式    │ Wasmサポート   │ 最適な用途      │
  ├─────────────┼──────────────┼───────────────┼────────────────┤
  │ tract       │ ONNX, TF     │ 優秀          │ 従来のモデル    │
  │ candle      │ safetensors  │ 良好          │ LLM, HFモデル  │
  │ burn        │ burn形式     │ 実験的         │ カスタム訓練    │
  │ ort (onnx)  │ ONNX         │ wasm-pack経由 │ ONNXエコシステム │
  └─────────────┴──────────────┴───────────────┴────────────────┘
```

## テンソル演算の詳細

あらゆるMLランタイムの中核はテンソル計算です。主要な演算を以下に示します：

### 行列乗算（GEMM）

ニューラルネットワークで最もパフォーマンスが重要な演算です：

```
  行列A (2x3)       行列B (3x2)       結果 (2x2)

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

### 活性化関数

```
  ReLU                    Sigmoid                  Softmax
  f(x) = max(0, x)       f(x) = 1/(1+e^-x)       f(xi) = e^xi / sum(e^xj)

  出力  │     ╱          出力  │    ──────       出力  │   ╱
        │    ╱                │  ╱              (確率) │  ╱
        │   ╱            0.5 │╱                      │╱────
  ───────┼──╱          ───────┼──────           ──────┼──────
        │               ╱   │                       │
        │             ╱     │                       │
  ───────┘                                          │
```

| 活性化関数  | 数式                   | 範囲       | 用途                        |
|-----------|----------------------|-----------|----------------------------|
| ReLU      | max(0, x)            | [0, inf)  | 隠れ層（デフォルト）           |
| Sigmoid   | 1 / (1 + e^-x)      | (0, 1)    | 二値分類                    |
| Softmax   | e^xi / sum(e^xj)    | (0, 1)    | 多クラス出力                 |
| Tanh      | (e^x - e^-x)/(e^x + e^-x) | (-1, 1) | RNN、隠れ層            |

## ブラウザでのモデルロード

```
  ┌─────────────────────────────────────────────────────┐
  │  モデルロードパイプライン                               │
  │                                                     │
  │  1. fetch("model.onnx")                             │
  │     │                                               │
  │     ▼                                               │
  │  2. ArrayBuffer（生バイト列）                         │
  │     │                                               │
  │     ▼                                               │
  │  3. Wasmに渡す: load_model(&bytes)                  │
  │     │                                               │
  │     ├── モデルグラフを解析（protobuf/flatbuf）        │
  │     ├── Wasmメモリに重みテンソルを確保                 │
  │     ├── グラフを最適化（演算融合、定数畳み込み）        │
  │     └── 不透明ハンドルを返す                          │
  │     │                                               │
  │     ▼                                               │
  │  4. 推論準備完了: predict(handle, input)              │
  └─────────────────────────────────────────────────────┘
```

```javascript
// JavaScript側
import init, { WasmModel } from './pkg/ml_wasm';

async function loadAndPredict() {
    await init();

    // モデルバイト列を取得
    const response = await fetch('/models/mobilenet_v2.onnx');
    const modelBytes = new Uint8Array(await response.arrayBuffer());

    // Wasmにロード
    const model = WasmModel.load(modelBytes);

    // 入力を準備（224x224 RGB画像をフラットなFloat32Arrayとして）
    const input = preprocessImage(imageElement);

    // 推論を実行
    const output = model.predict(input);
    const topClass = argmax(output);
    console.log(`予測: ${IMAGENET_CLASSES[topClass]}`);
}
```

## パフォーマンス: Wasm vs TensorFlow.js

一般的なノートPCでのベンチマーク（MobileNet V2、単一画像）：

```
  推論時間 (ms) — 低いほど良い

  TF.js (WebGL)  ████████████  12ms
  Wasm (tract)   ██████████████████  18ms
  TF.js (Wasm)   ████████████████████  20ms
  TF.js (CPU/JS) █████████████████████████████████████████  42ms

  ← 高速                                       低速 →
```

| ランタイム             | MobileNet V2 | ResNet50   | BERT (base) |
|----------------------|-------------|------------|-------------|
| TF.js WebGL          | 約12ms      | 約45ms     | 約120ms     |
| Wasm (tract)         | 約18ms      | 約80ms     | 約300ms     |
| TF.js Wasmバックエンド | 約20ms      | 約90ms     | 約350ms     |
| TF.js CPU (純JS)      | 約42ms      | 約200ms    | 約800ms     |
| ネイティブ (PyTorch CPU) | 約8ms     | 約30ms     | 約80ms      |

**主なポイント:**
- Wasmは純JavaScriptより**2-3倍高速**
- WebGL/WebGPUは大規模モデルではWasmに勝る（GPU並列処理）
- Wasmは**小規模モデル**でGPUセットアップのオーバーヘッドが支配的な場合に優位
- Wasmはより**予測可能**（GPUドライバのばらつきなし）

## Wasm ML vs WebGPUの使い分け

```
  判断マトリクス:

                     小規模モデル          大規模モデル
                     (< 1000万パラメータ)   (> 1億パラメータ)
  ┌─────────────────┬────────────────────┬────────────────────┐
  │ オフラインサポート │  Wasm ✓            │  Wasm（収まる場合）   │
  │ が必要？         │                    │  またはWebGPU       │
  ├─────────────────┼────────────────────┼────────────────────┤
  │ 安定した         │  Wasm ✓            │  Wasm              │
  │ パフォーマンス？  │  (GPUばらつきなし)   │ （遅いが安定）       │
  ├─────────────────┼────────────────────┼────────────────────┤
  │ 最大速度？       │  Wasm ✓            │  WebGPU ✓          │
  │                 │  (GPUオーバーヘッドが │  (GPU並列処理が      │
  │                 │   割に合わない)      │   優位)             │
  ├─────────────────┼────────────────────┼────────────────────┤
  │ ブラウザ互換性？  │  Wasm ✓            │  Wasm ✓            │
  │                 │  (ユニバーサル)      │  WebGPUはまだ新しい   │
  └─────────────────┴────────────────────┴────────────────────┘
```

## 実用的なユースケース

| アプリケーション              | モデル種類        | なぜWasmか?                         |
|----------------------------|-----------------|-------------------------------------|
| スパム検出                   | ロジスティック回帰 | 微小モデル、即時予測                    |
| 画像分類                    | MobileNet        | あらゆるデバイスで動作、オフライン対応    |
| テキスト自動補完              | 小規模RNN        | 低レイテンシ、プライバシー              |
| 姿勢推定（カメラ）            | PoseNet         | リアルタイム、サーバーとのラウンドトリップなし |
| ドキュメントOCR              | CRNN            | 機密文書がローカルに留まる              |
| 音声キーワード検出            | 小規模CNN        | 常時オン、低消費電力                    |
| 異常検知（IoT）              | オートエンコーダ   | エッジデバイス、通信不要                |

## まとめ

Wasmはフル精度、オフライン対応、サーバーコストゼロでML推論をブラウザにもたらします。ONNXモデルには`tract`を、Hugging Faceモデルには`candle`を使うか、Rustでカスタムテンソル演算を構築しましょう。Wasm MLは純JavaScriptの2-3倍高速で、GPUオーバーヘッドが割に合わない中小規模モデルに最適な選択肢です。大規模モデル（LLM、拡散モデル）の場合は、WasmとWebGPUを組み合わせてGPUアクセラレーションを活用しましょう。
