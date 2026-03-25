---
title: 状態管理
slug: state-management
difficulty: intermediate
tags: [data-structures]
order: 12
description: 構造体、共有メモリ、メッセージパッシングパターンを使って、Rust/WasmとJavaScript間で状態を共有・同期します。
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct AppState {
      items: Vec<String>,
      counter: u32,
      modified: bool,
  }

  #[wasm_bindgen]
  impl AppState {
      #[wasm_bindgen(constructor)]
      pub fn new() -> Self {
          Self {
              items: Vec::new(),
              counter: 0,
              modified: false,
          }
      }

      pub fn add_item(&mut self, item: &str) {
          self.items.push(item.to_string());
          self.counter += 1;
          self.modified = true;
      }

      pub fn remove_item(&mut self, index: usize) -> bool {
          if index < self.items.len() {
              self.items.remove(index);
              self.modified = true;
              true
          } else {
              false
          }
      }

      pub fn get_items(&self) -> String {
          self.items.join(",")
      }

      pub fn count(&self) -> u32 { self.counter }

      pub fn is_modified(&self) -> bool { self.modified }

      pub fn clear_modified(&mut self) { self.modified = false; }
  }
expected_output: |
  状態を作成
  追加: "apple" → カウント: 1
  追加: "banana" → カウント: 2
  アイテム: apple,banana
  変更済み: true
---

## 課題

Rust/Wasmアプリでは、状態は2つの場所に存在します：
- **Rust**（Wasmリニアメモリ） — ゲームの状態、データモデル、計算結果
- **JavaScript** — DOMの状態、UIの状態、ブラウザAPI

過剰なコピーなしにこれらを同期させるパターンが必要です。

## パターン1：Rustが状態を所有する（推奨）

最もシンプルで高性能なパターン — Rustがすべてのアプリケーション状態を所有し、JavaScriptは読み取りのみ行います：

```rust
#[wasm_bindgen]
pub struct GameState {
    score: u32,
    level: u32,
    entities: Vec<Entity>,
}

#[wasm_bindgen]
impl GameState {
    // JSはメソッドを呼び出して状態を変更
    pub fn update(&mut self, dt: f64) { /* 物理演算 */ }
    pub fn add_entity(&mut self, x: f64, y: f64) { /* ... */ }

    // JSはゲッターで状態を読み取り
    pub fn score(&self) -> u32 { self.score }
    pub fn level(&self) -> u32 { self.level }
    pub fn entity_positions(&self) -> Vec<f64> { /* フラット配列 */ }
}
```

```js
const state = new GameState();

function gameLoop() {
    state.update(16.67); // Rustで状態を変更
    render(state.entity_positions()); // JSから読み取り
    requestAnimationFrame(gameLoop);
}
```

## パターン2：メッセージパッシング

複雑なアプリケーションでは、コマンドパターンを使います — JSがアクションを送り、Rustがそれを処理します：

```rust
#[wasm_bindgen]
pub enum Action {
    AddItem,
    RemoveItem,
    UpdateScore,
    Reset,
}

#[wasm_bindgen]
impl AppState {
    pub fn dispatch(&mut self, action: Action, payload: &str) -> String {
        match action {
            Action::AddItem => {
                self.items.push(payload.to_string());
                format!("Added: {}", payload)
            }
            Action::RemoveItem => {
                // payloadからインデックスをパース
                "Removed".to_string()
            }
            Action::Reset => {
                self.items.clear();
                "Reset".to_string()
            }
            _ => "Unknown action".to_string(),
        }
    }
}
```

## パターン3：共有メモリ（上級）

大量データ（ゲームワールド、画像バッファ）の場合、メモリを直接共有します：

```rust
#[wasm_bindgen]
impl World {
    /// Wasmメモリ内のピクセルバッファへのポインタを返す
    pub fn pixels_ptr(&self) -> *const u8 {
        self.pixels.as_ptr()
    }

    pub fn pixels_len(&self) -> usize {
        self.pixels.len()
    }
}
```

```js
// JSがWasmメモリから直接読み取り — ゼロコピー！
const ptr = world.pixels_ptr();
const len = world.pixels_len();
const pixels = new Uint8Array(wasm.memory.buffer, ptr, len);

// Canvasに描画
const imageData = new ImageData(
    new Uint8ClampedArray(pixels.buffer, pixels.byteOffset, pixels.byteLength),
    width, height
);
ctx.putImageData(imageData, 0, 0);
```

## 避けるべきアンチパターン

| やってはいけないこと | 理由 | 代わりにすべきこと |
|---------------------|------|-------------------|
| 毎フレームJSONにシリアライズ | 遅い、文字列を確保する | フラット配列を返すか共有メモリを使用 |
| 状態全体をJSにクローン | コピーのコストが高い | JSが必要なもののゲッターを公開 |
| JSがRustメモリを直接変更 | 安全でなく、状態が破損する可能性 | バリデーション付きメソッドを使用 |
| RustにDOM参照を保持 | ライフタイムの問題が複雑 | JSがDOMを管理、Rustがデータを管理 |

## 試してみよう

スターターコードは、追加・削除・クエリメソッドを持つ状態構造体を示しています。この「Rustが状態を所有する」パターンは、ほとんどのアプリケーションで有効です。
