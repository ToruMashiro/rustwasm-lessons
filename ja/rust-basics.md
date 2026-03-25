---
title: Wasm向けRust基礎
slug: rust-basics
difficulty: beginner
tags: [getting-started]
order: 2
description: WebAssemblyに必要なRustの基本構文を学びます — 関数、型、構造体、エラーハンドリング。Rustの経験は不要です。
starter_code: |
  // === 関数と型 ===
  fn add(a: i32, b: i32) -> i32 {
      a + b  // セミコロンなし = 戻り値
  }

  // === 構造体 ===
  struct Point {
      x: f64,
      y: f64,
  }

  impl Point {
      fn new(x: f64, y: f64) -> Self {
          Self { x, y }
      }

      fn distance(&self, other: &Point) -> f64 {
          ((self.x - other.x).powi(2) + (self.y - other.y).powi(2)).sqrt()
      }
  }

  // === 文字列 ===
  fn greet(name: &str) -> String {
      format!("こんにちは、{}さん！", name)
  }

  // === OptionとResult ===
  fn safe_divide(a: f64, b: f64) -> Option<f64> {
      if b == 0.0 {
          None
      } else {
          Some(a / b)
      }
  }

  fn main() {
      // 関数
      println!("2 + 3 = {}", add(2, 3));

      // 構造体
      let p1 = Point::new(0.0, 0.0);
      let p2 = Point::new(3.0, 4.0);
      println!("距離: {}", p1.distance(&p2));

      // 文字列
      println!("{}", greet("Rustacean"));

      // Option
      match safe_divide(10.0, 3.0) {
          Some(result) => println!("10 / 3 = {:.2}", result),
          None => println!("ゼロで割ることはできません！"),
      }
  }
expected_output: |
  2 + 3 = 5
  距離: 5
  こんにちは、Rustaceanさん！
  10 / 3 = 3.33
---

## なぜRustの基礎を学ぶのか？

WebAssemblyを使うためにRustのすべてをマスターする必要はありません。このレッスンでは、すべてのWasmプロジェクトで使う**最小限のサブセット**をカバーします。

## 関数

Rust関数は `fn` キーワード、明示的な型、そして最後の式（`;` なし）が戻り値になります：

```rust
fn multiply(a: i32, b: i32) -> i32 {
    a * b  // 暗黙的に返される
}
```

## インポート: `use`、`::`、`*`

すべてのWasmファイルの先頭にこの行があります：

```rust
use wasm_bindgen::prelude::*;
```

一つずつ分解して説明します：

### `use` — アイテムをスコープに取り込む

`use` はJavaScriptの `import` に相当します。これがないと、毎回フルパスを書く必要があります：

```rust
// useなし: 冗長
wasm_bindgen::prelude::wasm_bindgen

// useあり: 簡潔
use wasm_bindgen::prelude::*;
#[wasm_bindgen]  // 直接使える
```

### `::` — パス区切り

`::` はモジュールをたどります（ファイルパスの `/` やJavaScriptの `.` と同じ）：

```rust
// クレート名::モジュール::サブモジュール::アイテム
use wasm_bindgen::prelude::*;
//  ^^^^^^^^^^^^   ^^^^^^^   ^
//  クレート名      モジュール  その中のすべて

use std::collections::HashMap;
//  ^^^  ^^^^^^^^^^^  ^^^^^^^
//  std   モジュール    特定の型

use web_sys::Document;
//  ^^^^^^^  ^^^^^^^^
//  クレート  特定の型
```

`クレート名::モジュール::アイテム` と考えてください。

### `*` — グロブインポート（すべて）

`*` はモジュールのすべての公開アイテムをインポートします：

```rust
use wasm_bindgen::prelude::*;  // preludeのすべてをインポート
```

これは「prelude」モジュールでよく使われます — 最もよく使うアイテムをまとめたものです。`wasm_bindgen::prelude::*` で以下が使えるようになります：
- `#[wasm_bindgen]` アトリビュート
- `JsValue` 型
- `JsCast` トレイト（`dyn_into()` 用）
- その他の必須アイテム

特定のアイテムだけをインポートすることもできます：

```rust
// 必要なものだけをインポート
use wasm_bindgen::prelude::wasm_bindgen;
use wasm_bindgen::JsValue;

// 波括弧でグループ化
use web_sys::{Document, Element, Window};
//            ^^^^^^^^  ^^^^^^^  ^^^^^^
//            web_sysから3つの型を指定
```

### Wasmでよく使うインポートパターン

```rust
// 必須（ほぼすべてのファイルで）
use wasm_bindgen::prelude::*;

// 特定のWeb API
use web_sys::{Document, Element, HtmlCanvasElement};

// JavaScript組み込み型
use js_sys::{Array, Object, JSON};

// イベントハンドラ用クロージャ
use wasm_bindgen::closure::Closure;

// 非同期サポート
use wasm_bindgen_futures::JsFuture;
```

## Wasm型の完全リファレンス

`wasm-bindgen` を通じてRustとJavaScript間でやり取りできるすべての型を示します：

### プリミティブ（ゼロコスト — Wasmを通じて直接渡される）

| Rust | JavaScript | 備考 |
|------|-----------|------|
| `i8`, `i16`, `i32` | `number` | 符号付き整数 |
| `u8`, `u16`, `u32` | `number` | 符号なし整数 |
| `i64`, `u64` | `BigInt` | 64ビット整数は `number` ではなく `BigInt` に対応 |
| `isize`, `usize` | `number` | プラットフォームサイズの整数（Wasmでは32ビット） |
| `f32`, `f64` | `number` | 浮動小数点 |
| `bool` | `boolean` | |
| `char` | `string` | 単一のUnicode文字 |

### 文字列（Wasm境界を越えてコピーされる）

| Rust | JavaScript | 方向 | 備考 |
|------|-----------|------|------|
| `&str` | `string` | JS → Rust | 借用、読み取り専用。JS文字列がUTF-8にエンコードされる |
| `String` | `string` | Rust → JS | 所有。UTF-8バイトがJS文字列にデコードされる |

### 型付き配列（コピーまたはビューによるゼロコピー）

| Rust | JavaScript | 備考 |
|------|-----------|------|
| `Vec<u8>`, `&[u8]` | `Uint8Array` | バイトバッファ、ファイルデータ |
| `Vec<i8>`, `&[i8]` | `Int8Array` | |
| `Vec<u16>`, `&[u16]` | `Uint16Array` | |
| `Vec<i16>`, `&[i16]` | `Int16Array` | |
| `Vec<u32>`, `&[u32]` | `Uint32Array` | |
| `Vec<i32>`, `&[i32]` | `Int32Array` | |
| `Vec<f32>`, `&[f32]` | `Float32Array` | グラフィックス、音声データ |
| `Vec<f64>`, `&[f64]` | `Float64Array` | 物理シミュレーション |
| `Vec<u64>`, `&[u64]` | `BigUint64Array` | |
| `Vec<i64>`, `&[i64]` | `BigInt64Array` | |

### 複合型

| Rust | JavaScript | 備考 |
|------|-----------|------|
| `JsValue` | `any` | あらゆるJS値のキャッチオール |
| `Option<T>` | `T \| undefined` | `None` は `undefined` になる |
| `Result<T, JsValue>` | `T`（`Err` 時にスロー） | エラーはJS例外になる |
| `#[wasm_bindgen] struct` | `class` | JSクラスとしてエクスポート、ポインタベース（コピーなし） |
| `Box<[JsValue]>` | `Array` | 任意の値のJS配列 |
| `Closure<dyn FnMut()>` | `Function` | RustクロージャをJSコールバックとして使用 |

### js-sys 組み込み型

| Rust (`js_sys::`) | JavaScript | 備考 |
|-------------------|-----------|------|
| `js_sys::Array` | `Array` | 完全なArray API |
| `js_sys::Object` | `Object` | 汎用JSオブジェクト |
| `js_sys::Map` | `Map` | |
| `js_sys::Set` | `Set` | |
| `js_sys::Date` | `Date` | |
| `js_sys::RegExp` | `RegExp` | |
| `js_sys::Promise` | `Promise` | `wasm-bindgen-futures` と併用 |
| `js_sys::Function` | `Function` | 呼び出し可能なJS関数 |
| `js_sys::Uint8Array` | `Uint8Array` | Wasmメモリへのゼロコピービュー |

## 構造体

構造体はRust版のクラスです。これをJavaScriptにWasmオブジェクトとしてエクスポートします：

```rust
struct GameState {
    score: u32,
    level: u32,
}

impl GameState {
    fn new() -> Self {
        Self { score: 0, level: 1 }
    }

    fn add_points(&mut self, points: u32) {
        self.score += points;
    }
}
```

## 所有権と借用（簡略版）

Rustで最もつまずきやすい概念です。Wasmでは以下を知っておけば十分です：

- **`&str`** — 文字列を借用（読み取り専用、低コスト）
- **`String`** — 文字列を所有（変更可能、メモリ確保が必要）
- **`&self`** — メソッドが構造体を借用（読み取り専用）
- **`&mut self`** — メソッドが構造体を借用（変更可能）

```rust
fn print_name(name: &str) {     // 借用、所有しない
    println!("名前: {}", name);
}

fn make_greeting(name: &str) -> String {  // 所有されたStringを返す
    format!("こんにちは、{}さん！", name)
}
```

## エラーハンドリング：OptionとResult

失敗する可能性のあるWasm関数は `Result<T, JsValue>` を返すべきです：

```rust
// Option: 値が存在しない可能性
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".to_string()) } else { None }
}

// Result: 操作が失敗する可能性（Wasmではエラーにはsを使用）
fn parse_number(s: &str) -> Result<i32, String> {
    s.parse::<i32>().map_err(|e| e.to_string())
}
```

## 試してみよう

**実行** をクリックしてすべての例を動かしてみましょう。コードを変更してみてください — `Point` に新しいメソッドを追加したり、`greet` 関数を変更してみましょう。
