---
title: WebAssemblyとは？
slug: what-is-wasm
difficulty: beginner
tags: [getting-started]
order: 1
description: WebAssemblyとは何か、なぜ存在するのか、Rustがどう関わるのかを理解しましょう。コードを書く前に全体像を把握します。
starter_code: |
  // これは通常のRustコードです。まだWasm固有のコードはありません。
  // ネイティブでもWasmでも同じように動作します。

  fn main() {
      let languages = ["C", "C++", "Rust", "Go", "AssemblyScript"];

      println!("WebAssemblyにコンパイルできる言語:");
      for (i, lang) in languages.iter().enumerate() {
          println!("  {}. {}", i + 1, lang);
      }

      println!("\nRustが最も人気な理由:");
      println!("  - ガベージコレクタなし（小さなバイナリ）");
      println!("  - ランタイムコストなしのメモリ安全性");
      println!("  - wasm-packによるファーストクラスのWasmサポート");
  }
expected_output: |
  WebAssemblyにコンパイルできる言語:
    1. C
    2. C++
    3. Rust
    4. Go
    5. AssemblyScript

  Rustが最も人気な理由:
    - ガベージコレクタなし（小さなバイナリ）
    - ランタイムコストなしのメモリ安全性
    - wasm-packによるファーストクラスのWasmサポート
---

## WebAssemblyとは？

WebAssembly（Wasm）は、ブラウザでネイティブに近い速度で実行されるバイナリ命令形式です。ポータブルなコンパイルターゲットと考えてください — Rust、C++、Goでコードを書き、`.wasm` にコンパイルし、ブラウザがJavaScriptと一緒に実行します。

## なぜ存在するのか？

JavaScriptは強力ですが、限界があります：

- **CPU集約型タスク**（暗号、画像処理、物理演算）はJSでは遅い
- **大規模コードベース**（ゲーム、CADツール）には予測可能なパフォーマンスが必要
- **既存のコード**（C/C++/Rust）はJSに移植しないとブラウザで動かない

Wasmはこの3つすべてを解決します。JavaScriptの代替ではなく、補完です。

## 仕組み

```
┌──────────┐    コンパイル    ┌──────────┐     実行      ┌──────────┐
│  Rust    │ ──────────────▶ │  .wasm   │ ────────────▶ │ ブラウザ  │
│  ソース  │   (wasm-pack)   │ バイナリ  │  (JSインポート) │  エンジン │
└──────────┘                 └──────────┘               └──────────┘
```

1. Rustコードを書く
2. `wasm-pack` が `.wasm` バイナリ + JSグルーコードにコンパイル
3. JavaScriptがRust関数をインポートして呼び出す
4. ブラウザがWasmをネイティブに近い速度で実行

## なぜRustなのか？

多くの言語がWasmをターゲットにできますが、Rustが最適です：

- **ガベージコレクタなし** — Wasmバイナリが小さい（多くの場合50KB未満）
- **メモリ安全性** — コンパイラがビルド時にバグを検出
- **ゼロコスト抽象化** — 高レベルコードが高速な機械語にコンパイル
- **ファーストクラスのツール** — `wasm-pack`、`wasm-bindgen`、`web-sys` が成熟

## Wasmにできないこと（まだ）

- DOM直接アクセス不可 — JavaScriptバインディングを経由する必要あり
- デフォルトではスレッドなし — `SharedArrayBuffer` はオプトイン
- ファイルシステムアクセス不可 — サンドボックス環境で実行
- JavaScriptを置き換えられない — 補完するもの

## 試してみよう

右のコードは通常のRustです。**実行** をクリックして動かしてみましょう。次のレッスンでは、このコードをWebAssemblyを通じて *ブラウザ内で* 実行する方法を学びます。
