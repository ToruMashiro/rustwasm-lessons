---
title: パーティクルシミュレーション
slug: particle-simulation
difficulty: advanced
tags: [simulation, graphics]
order: 8
description: Rust/Wasmで駆動する高性能パーティクルシステムをHTML Canvas上にレンダリングします。
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct Particle {
      x: f64,
      y: f64,
      vx: f64,
      vy: f64,
      radius: f64,
  }

  #[wasm_bindgen]
  pub struct Simulation {
      particles: Vec<Particle>,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Simulation {
      #[wasm_bindgen(constructor)]
      pub fn new(width: f64, height: f64, count: usize) -> Self {
          let mut particles = Vec::with_capacity(count);
          for i in 0..count {
              let angle = (i as f64) * 2.0 * std::f64::consts::PI / (count as f64);
              particles.push(Particle {
                  x: width / 2.0 + (width / 4.0) * angle.cos(),
                  y: height / 2.0 + (height / 4.0) * angle.sin(),
                  vx: angle.cos() * 2.0,
                  vy: angle.sin() * 2.0,
                  radius: 3.0,
              });
          }
          Self { particles, width, height }
      }

      pub fn tick(&mut self) {
          for p in &mut self.particles {
              p.x += p.vx;
              p.y += p.vy;

              // 壁で跳ね返る
              if p.x - p.radius < 0.0 || p.x + p.radius > self.width {
                  p.vx = -p.vx;
              }
              if p.y - p.radius < 0.0 || p.y + p.radius > self.height {
                  p.vy = -p.vy;
              }
          }
      }

      pub fn positions(&self) -> Vec<f64> {
          self.particles.iter()
              .flat_map(|p| vec![p.x, p.y, p.radius])
              .collect()
      }

      pub fn particle_count(&self) -> usize {
          self.particles.len()
      }
  }
expected_output: |
  シミュレーション開始: 500個のパーティクル
  フレーム1: 全パーティクル更新完了
  requestAnimationFrameにより約60fpsでレンダリング中
---

## はじめに

ここがRust/Wasmが真価を発揮する場面です — フレームごとに数千の物理計算をネイティブに近い速度で実行し、JavaScriptはCanvasレンダリングのみを担当します。この分離により、10,000以上のパーティクルでも60fpsを実現できます。

## アーキテクチャ

```
┌─────────────┐    positions()    ┌──────────────┐
│  Rust/Wasm   │ ───────────────▶ │  JavaScript  │
│  シミュレーション │                  │  Canvas 2D   │
│  （物理演算）  │ ◀─────────────── │ （レンダリング） │
└─────────────┘    tick()         └──────────────┘
```

重要なポイント：**Rustがシミュレーション状態を所有します。** JavaScriptは位置データを読み取ってCanvasに描画するだけです。これにより、毎フレームの高コストなデータシリアライゼーションを回避できます。

## セットアップ

```toml
[dependencies]
wasm-bindgen = "0.2"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "HtmlCanvasElement",
    "CanvasRenderingContext2d",
]
```

## データ転送：フラット配列

RustからJSへ大量データを送る最も効率的な方法はフラットな `Vec<f64>` です：

```rust
// Vec<Particle>を返す代わりに（Wasm境界を越えられない）、
// フラット配列を返す：[x0, y0, r0, x1, y1, r1, ...]
pub fn positions(&self) -> Vec<f64> {
    self.particles.iter()
        .flat_map(|p| vec![p.x, p.y, p.radius])
        .collect()
}
```

JavaScriptでは、これが `Float64Array` になります — JSONパースなしで直接イテレート可能：

```js
const positions = simulation.positions();
for (let i = 0; i < positions.length; i += 3) {
    const x = positions[i];
    const y = positions[i + 1];
    const radius = positions[i + 2];
    // 描画...
}
```

## JavaScriptレンダリングループ

```js
import init, { Simulation } from './pkg/particles.js';

async function run() {
    await init();

    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const sim = new Simulation(canvas.width, canvas.height, 1000);

    function frame() {
        // Rustで物理演算
        sim.tick();

        // 位置を読み取り
        const positions = sim.positions();

        // JSでレンダリング
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = '#654ff0';

        for (let i = 0; i < positions.length; i += 3) {
            ctx.beginPath();
            ctx.arc(positions[i], positions[i+1], positions[i+2], 0, Math.PI * 2);
            ctx.fill();
        }

        requestAnimationFrame(frame);
    }

    requestAnimationFrame(frame);
}

run();
```

## パフォーマンス最適化のヒント

### 1. バッファの再利用 — 毎フレームのメモリ確保を避ける

```rust
pub struct Simulation {
    particles: Vec<Particle>,
    position_buffer: Vec<f64>,  // これを再利用
    // ...
}

pub fn positions(&mut self) -> Vec<f64> {
    self.position_buffer.clear();
    for p in &self.particles {
        self.position_buffer.push(p.x);
        self.position_buffer.push(p.y);
        self.position_buffer.push(p.radius);
    }
    self.position_buffer.clone()
}
```

### 2. SharedArrayBufferでゼロコピー（上級）

毎フレームデータをコピーする代わりに、RustとJS間でメモリを共有します：

```rust
// パーティクルデータへのポインタを公開
pub fn positions_ptr(&self) -> *const f64 { /* ... */ }
pub fn positions_len(&self) -> usize { /* ... */ }
```

```js
// JSがWasmメモリから直接読み取り（ゼロコピー！）
const ptr = sim.positions_ptr();
const len = sim.positions_len();
const view = new Float64Array(wasm.memory.buffer, ptr, len);
```

### 3. 衝突検出のための空間分割

パーティクル同士の衝突にはO(n^2)チェックを避けるため、グリッドやクアッドツリーを使用します：

```rust
pub fn tick_with_collisions(&mut self) {
    // グリッドベースのブロードフェーズ
    let cell_size = 20.0;
    let mut grid: HashMap<(i32, i32), Vec<usize>> = HashMap::new();

    for (i, p) in self.particles.iter().enumerate() {
        let key = ((p.x / cell_size) as i32, (p.y / cell_size) as i32);
        grid.entry(key).or_default().push(i);
    }

    // 同じセルまたは隣接セルのパーティクルのみチェック
    // ...
}
```

## パフォーマンスベンチマーク

| パーティクル数 | 純JS (fps) | Rust/Wasm (fps) |
|--------------|-----------|----------------|
| 1,000 | 60 | 60 |
| 5,000 | 35 | 60 |
| 10,000 | 15 | 60 |
| 50,000 | 3 | 45 |

複雑な物理演算（衝突、力、制約）があると、差は劇的に広がります。

## 試してみよう

重力、パーティクル同士の衝突、またはマウスインタラクションをシミュレーションに追加してみましょう。
