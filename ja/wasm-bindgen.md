---
title: wasm-bindgenの仕組み
slug: wasm-bindgen
difficulty: beginner
tags: [getting-started]
order: 4
description: RustとJavaScriptの橋渡し — wasm-bindgenが型変換、関数エクスポート、グルーコード生成をどう行うかを理解します。
starter_code: |
  use wasm_bindgen::prelude::*;

  // === JavaScriptへのエクスポート ===

  // シンプルな関数: プリミティブは直接渡せる
  #[wasm_bindgen]
  pub fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  // 文字列変換: &str (JSから) → String (JSへ)
  #[wasm_bindgen]
  pub fn greet(name: &str) -> String {
      format!("こんにちは、{}さん！", name)
  }

  // 構造体をJSクラスとしてエクスポート
  #[wasm_bindgen]
  pub struct Counter {
      count: u32,
  }

  #[wasm_bindgen]
  impl Counter {
      #[wasm_bindgen(constructor)]
      pub fn new() -> Self {
          Self { count: 0 }
      }

      pub fn increment(&mut self) {
          self.count += 1;
      }

      pub fn value(&self) -> u32 {
          self.count
      }
  }

  // === JavaScriptからのインポート ===

  #[wasm_bindgen]
  extern "C" {
      #[wasm_bindgen(js_namespace = console)]
      fn log(s: &str);

      #[wasm_bindgen(js_namespace = Math)]
      fn random() -> f64;
  }
expected_output: |
  // JavaScriptから:
  import init, { add, greet, Counter } from './pkg/my_wasm.js';

  await init();

  add(2, 3);           // → 5
  greet("World");      // → "こんにちは、Worldさん！"

  const c = new Counter();
  c.increment();
  c.value();           // → 1
---

## wasm-bindgenとは？

`wasm-bindgen` はRustとJavaScriptの橋渡しです。WebAssemblyはネイティブで4つの数値型（`i32`、`i64`、`f32`、`f64`）のみをサポートします。それ以外 — 文字列、構造体、配列、オブジェクト — は変換が必要です。`wasm-bindgen` がこの変換を自動的に行うグルーコードを生成します。

## `#[wasm_bindgen]` アトリビュート

この1つのアトリビュートが、エクスポートやインポートの対象を `wasm-bindgen` に伝えます：

```rust
#[wasm_bindgen]          // 関数: JSにエクスポート
pub fn my_func() {}

#[wasm_bindgen]          // 構造体: JSクラスを作成
pub struct MyStruct {}

#[wasm_bindgen]          // implブロック: メソッドをエクスポート
impl MyStruct {}

#[wasm_bindgen]          // extern "C": JSからインポート
extern "C" {
    fn alert(s: &str);
}
```

## 型が境界を越える仕組み

| Rust | JavaScript | コスト | 仕組み |
|------|-----------|--------|--------|
| `i32`, `f64` | `number` | 無料 | ネイティブWasm数値型 |
| `i64`, `u64` | `BigInt` | 無料 | Wasm i64がBigIntに対応 |
| `bool` | `boolean` | 無料 | Wasmではi32（0/1）として表現 |
| `&str` | `string` | コピー | JSが文字列をUTF-8にエンコードしてWasmリニアメモリに格納 |
| `String` | `string` | コピー | RustがUTF-8バイトを書き込み、JSがデコード |
| `&[u8]` | `Uint8Array` | コピー | JSとWasmメモリ間でバイトをコピー |
| `Vec<f64>` | `Float64Array` | コピー | 大量の数値データ転送 |
| `Option<T>` | `T \| undefined` | 可変 | `None` はJSで `undefined` になる |
| `Result<T, E>` | `T` またはスロー | 可変 | `Err` はJS例外になる |
| `JsValue` | `any` | 参照 | 任意のJS値への不透明なハンドル |
| `#[wasm_bindgen] struct` | `class` | ポインタ | JSがWasmメモリ上のポインタを保持 |
| `Closure<dyn FnMut()>` | `Function` | メモリ確保 | RustクロージャをJSコールバックとしてラップ |

**重要なポイント**: プリミティブは無料。文字列はコピー。構造体はポインタ。クロージャはメモリ確保が必要。

## wasm-packが生成するもの

`wasm-pack build` を実行すると以下が生成されます：

```
pkg/
├── my_wasm_bg.wasm       # コンパイル済みWasmバイナリ
├── my_wasm_bg.wasm.d.ts  # Wasmバイナリ用TypeScript型定義
├── my_wasm.js            # JavaScriptグルーコード（型変換を処理）
├── my_wasm.d.ts          # 公開API用TypeScript型定義
└── package.json          # npmに公開可能なパッケージ
```

`.js` グルーファイルが処理するもの：
- `.wasm` バイナリの読み込みと初期化
- JS（UTF-16）とWasm（UTF-8）間の文字列エンコード/デコード
- 文字列と配列のWasmメモリ割り当て管理
- エクスポートされた構造体の解放後使用の防止
- 完全な型安全性を備えたTypeScriptフレンドリーなAPIの提供

## JavaScript関数のインポート

`extern "C"` ブロックを使って、RustからあらゆるJavaScript APIを呼び出せます：

```rust
#[wasm_bindgen]
extern "C" {
    // window.alert()
    fn alert(s: &str);

    // console.log() — JS名前空間を指定
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    // リネーム：Rust名とJS名を異なるものに
    #[wasm_bindgen(js_name = "setTimeout")]
    fn set_timeout(closure: &Closure<dyn FnMut()>, ms: u32);

    // グローバル変数のインポート
    #[wasm_bindgen(js_name = "document")]
    static DOCUMENT: web_sys::Document;
}
```

## よく使うアトリビュートパターン

### コンストラクタ — JSで `new MyStruct()` を使えるようにする

```rust
#[wasm_bindgen(constructor)]
pub fn new() -> Self { Self { count: 0 } }
```

### ゲッター / セッター — JSで `obj.name` を使えるようにする

```rust
#[wasm_bindgen(getter)]
pub fn name(&self) -> String { self.name.clone() }

#[wasm_bindgen(setter)]
pub fn set_name(&mut self, name: &str) { self.name = name.to_string(); }
```

### js_name — JS規約に合わせてリネーム

```rust
#[wasm_bindgen(js_name = "calculateTotal")]
pub fn calculate_total(&self) -> f64 { /* ... */ }
```

### skip — publicメソッドをJSから隠す

```rust
#[wasm_bindgen(skip)]
pub fn internal_method(&self) { /* エクスポートされない */ }
```

## Cargo.tomlの設定

すべてのRust/Wasmプロジェクトに必要な依存関係：

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
# オプション: ブラウザAPIバインディング
web-sys = { version = "0.3", features = ["Window", "Document"] }
# オプション: JS組み込み型
js-sys = "0.3"
# オプション: async/awaitサポート
wasm-bindgen-futures = "0.4"
```

## 試してみよう

スターターコードは3つの主要パターンを示しています：関数のエクスポート、構造体のエクスポート、JS関数のインポート。この3つのパターンで実際のWasm開発の90%をカバーできます。
