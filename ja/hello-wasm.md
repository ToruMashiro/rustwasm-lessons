---
title: はじめてのWebAssembly
slug: hello-wasm
difficulty: beginner
tags: [getting-started]
order: 3
description: 初めてのRust + WebAssemblyプロジェクト。wasm-bindgenがRustとJavaScriptをどう繋ぐかを学びます。
starter_code: |
  use wasm_bindgen::prelude::*;

  // JavaScriptに関数をエクスポート
  #[wasm_bindgen]
  pub fn greet(name: &str) -> String {
      format!("こんにちは、{}さん！WebAssemblyへようこそ。", name)
  }

  // シンプルな計算関数をエクスポート
  #[wasm_bindgen]
  pub fn add(a: i32, b: i32) -> i32 {
      a + b
  }
expected_output: |
  こんにちは、World さん！WebAssemblyへようこそ。
  2 + 3 = 5
---

## はじめに

WebAssembly（Wasm）は、コンパイルされたコードをネイティブに近い速度でブラウザ上で実行できる技術です。Rustはゼロコスト抽象化、ガベージコレクタなし、小さなバイナリサイズという特徴から、Wasm開発に最適な言語の一つです。

## 仕組み

1. Rustコードを書き、関数に `#[wasm_bindgen]` アトリビュートを付けます
2. `wasm-pack` ツールがRustコードを `.wasm` バイナリにコンパイルします
3. JavaScriptからエクスポートされた関数を直接呼び出せます

## 主要な概念

- **`#[wasm_bindgen]`** — JavaScriptからアクセス可能な関数を示すアトリビュート
- **`&str`** — Rustの文字列スライス型。JS文字列から自動変換されます
- **戻り値の型** — `String`、`i32`、`f64` などのRust型はJS型に自動変換されます

## JavaScriptからの呼び出し

`wasm-pack` でコンパイルした後、Rust関数を直接インポートして呼び出せます:

```js
import init, { greet, add } from './pkg/hello_wasm.js';

async function run() {
    await init();
    console.log(greet("World"));
    console.log("2 + 3 =", add(2, 3));
}
run();
```

## なぜ2つのインポート？ `init` と `{ greet, add }`

すべてのWasmモジュールには**2種類のインポート**があります：

```js
import init, { greet, add } from './pkg/hello_wasm.js';
//     ^^^^  ^^^^^^^^^^^^^^
//     │     └── 名前付きエクスポート: あなたのRust関数と構造体
//     └── デフォルトエクスポート: Wasm初期化関数
```

**`init`**（デフォルトエクスポート）：
- `.wasm` バイナリファイルをダウンロードしてコンパイル
- Rust関数を使う前に**一度だけ**呼び出す必要がある
- `Promise` を返す — `await init()` または `.then()` で使用
- この呼び出し後、すべての名前付きエクスポートが使える

**`{ greet, add }`**（名前付きエクスポート）：
- `#[wasm_bindgen] pub fn` で定義した関数
- `#[wasm_bindgen] pub struct` で定義した型（JSクラスとして）
- `init()` が完了するまで**使用できない**

そのため、Wasmコードは常にこのパターンに従います：

```js
// ステップ1: 初期化（.wasmをダウンロード＆コンパイル）
await init();

// ステップ2: Rust関数を使える
greet("World");    // ✓ 動作する
add(2, 3);         // ✓ 動作する
```

`init()` の前に関数を呼ぶと：

```js
greet("World");    // ✗ エラー: Wasmが初期化されていない
await init();
greet("World");    // ✓ これで動作する
```

## 試してみよう

`greet` 関数を変更してカスタムメッセージを追加したり、`multiply` のような新しい関数を作ってみましょう。
