---
title: Wasmゲームの構築
slug: wasm-game
difficulty: advanced
tags: [graphics, simulation]
order: 24
description: Rust/Wasmで完全なブラウザゲームを構築する — ゲームループ、入力処理、衝突判定、Canvas描画。
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct Game {
      player_x: f64,
      player_y: f64,
      player_size: f64,
      targets: Vec<(f64, f64)>,
      score: u32,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Game {
      #[wasm_bindgen(constructor)]
      pub fn new(width: f64, height: f64) -> Self {
          let mut targets = Vec::new();
          for i in 0..5 {
              targets.push((
                  100.0 + (i as f64) * 120.0,
                  100.0 + (i as f64) * 60.0,
              ));
          }
          Self {
              player_x: width / 2.0,
              player_y: height / 2.0,
              player_size: 20.0,
              targets,
              score: 0,
              width,
              height,
          }
      }

      pub fn move_player(&mut self, dx: f64, dy: f64) {
          self.player_x = (self.player_x + dx).clamp(0.0, self.width);
          self.player_y = (self.player_y + dy).clamp(0.0, self.height);
      }

      pub fn update(&mut self) {
          // ターゲットとの衝突判定
          self.targets.retain(|&(tx, ty)| {
              let dist = ((self.player_x - tx).powi(2) + (self.player_y - ty).powi(2)).sqrt();
              if dist < self.player_size + 15.0 {
                  self.score += 10;
                  false // 取得したターゲットを削除
              } else {
                  true
              }
          });

          // すべて取得したらターゲットを再生成
          if self.targets.is_empty() {
              for i in 0..5 {
                  self.targets.push((
                      (i as f64 * 137.0) % self.width,
                      (i as f64 * 97.0) % self.height,
                  ));
              }
          }
      }

      pub fn score(&self) -> u32 { self.score }
      pub fn player_x(&self) -> f64 { self.player_x }
      pub fn player_y(&self) -> f64 { self.player_y }
      pub fn player_size(&self) -> f64 { self.player_size }

      pub fn target_count(&self) -> usize { self.targets.len() }

      pub fn target_positions(&self) -> Vec<f64> {
          self.targets.iter().flat_map(|&(x, y)| vec![x, y]).collect()
      }
  }
expected_output: |
  Game created: 800x600
  Player at (400, 300)
  5 targets spawned
  Score: 0
  Collecting targets...
  Score: 50
---

## ゲームアーキテクチャ

Wasmゲームは責任をRustとJavaScriptに分担します:

```
Rust（ゲームロジック）             JavaScript（描画 + 入力）
┌────────────────────┐         ┌────────────────────────┐
│ ゲーム状態          │         │ Canvas 2D描画          │
│ 物理/衝突判定       │◀───────▶│ キーボード/マウス入力    │
│ エンティティ管理     │  data   │ requestAnimationFrame  │
│ スコア/ルール       │         │ オーディオ再生          │
└────────────────────┘         └────────────────────────┘
```

**Rustが状態を管理し、JSがI/Oを担当します。**

## ゲームループ

```js
import init, { Game } from './pkg/game.js';

await init();

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const game = new Game(canvas.width, canvas.height);

// 押されているキーを追跡
const keys = new Set();
document.addEventListener('keydown', (e) => keys.add(e.key));
document.addEventListener('keyup', (e) => keys.delete(e.key));

function gameLoop() {
    // 1. 入力
    const speed = 4;
    if (keys.has('ArrowLeft'))  game.move_player(-speed, 0);
    if (keys.has('ArrowRight')) game.move_player(speed, 0);
    if (keys.has('ArrowUp'))    game.move_player(0, -speed);
    if (keys.has('ArrowDown'))  game.move_player(0, speed);

    // 2. 更新（Rust）
    game.update();

    // 3. 描画（JS）
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // プレイヤーを描画
    ctx.fillStyle = '#654ff0';
    ctx.beginPath();
    ctx.arc(game.player_x(), game.player_y(), game.player_size(), 0, Math.PI * 2);
    ctx.fill();

    // ターゲットを描画
    const positions = game.target_positions();
    ctx.fillStyle = '#ce422b';
    for (let i = 0; i < positions.length; i += 2) {
        ctx.beginPath();
        ctx.arc(positions[i], positions[i + 1], 15, 0, Math.PI * 2);
        ctx.fill();
    }

    // スコアを描画
    ctx.fillStyle = '#f1f5f9';
    ctx.font = '20px monospace';
    ctx.fillText(`Score: ${game.score()}`, 10, 30);

    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

## 衝突判定

### 円同士の衝突

```rust
fn circles_collide(x1: f64, y1: f64, r1: f64, x2: f64, y2: f64, r2: f64) -> bool {
    let dx = x1 - x2;
    let dy = y1 - y2;
    let dist_sq = dx * dx + dy * dy;
    let radii = r1 + r2;
    dist_sq < radii * radii  // パフォーマンスのためsqrtを回避
}
```

### AABB（矩形）衝突

```rust
fn aabb_collide(
    x1: f64, y1: f64, w1: f64, h1: f64,
    x2: f64, y2: f64, w2: f64, h2: f64,
) -> bool {
    x1 < x2 + w2 && x1 + w1 > x2 && y1 < y2 + h2 && y1 + h1 > y2
}
```

## エンティティ・コンポーネント・システム（ECS）パターン

複雑なゲームにはECSパターンを使います:

```rust
#[wasm_bindgen]
pub struct World {
    positions: Vec<f64>,    // [x0, y0, x1, y1, ...]
    velocities: Vec<f64>,   // [vx0, vy0, vx1, vy1, ...]
    sizes: Vec<f64>,        // [r0, r1, ...]
    alive: Vec<bool>,
    count: usize,
}

#[wasm_bindgen]
impl World {
    pub fn update(&mut self, dt: f64) {
        for i in 0..self.count {
            if !self.alive[i] { continue; }
            let idx = i * 2;
            self.positions[idx] += self.velocities[idx] * dt;
            self.positions[idx + 1] += self.velocities[idx + 1] * dt;
        }
    }

    pub fn positions(&self) -> Vec<f64> {
        self.positions.clone()
    }
}
```

## ゲームのパフォーマンスヒント

| ヒント | 理由 |
|-----|-----|
| バルクデータにはフラット配列（`Vec<f64>`）を使う | Wasm↔JS間の境界越えを最小化 |
| フレームごとのアロケーションを避ける | `.clear()` でバッファを再利用 |
| JS向けコードでは `f32` ではなく `f64` を使う | JSの数値はf64 |
| 空間分割で衝突判定する | グリッドや四分木でO(n²)をO(n)に |
| ゲームロジックはRust、描画はJSで | Canvas APIはWasmからアクセスできない |

## 試してみよう

スターターコードはシンプルな収集ゲームを実装しています — プレイヤーが移動してターゲットを集めます。実際のプロジェクトでは、キーボード入力とCanvasの描画ループに接続します。
