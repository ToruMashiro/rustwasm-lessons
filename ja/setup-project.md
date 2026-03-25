---
title: 初めてのプロジェクトセットアップ
slug: setup-project
difficulty: beginner
tags: [getting-started]
order: 9
description: wasm-packを使ってRust/Wasmプロジェクトをゼロから作成し、Cargo.tomlを設定して、JavaScriptバンドラーと統合します。
starter_code: |
  fn main() {
      println!("=== プロジェクトセットアップチェックリスト ===");
      println!();

      let steps = [
          ("Install Rust", "curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh"),
          ("Install wasm-pack", "cargo install wasm-pack"),
          ("Create project", "cargo new --lib my-wasm-app"),
          ("Configure Cargo.toml", "crate-type = [\"cdylib\"]"),
          ("Build", "wasm-pack build --target web"),
      ];

      for (i, (step, cmd)) in steps.iter().enumerate() {
          println!("  {}. {} ", i + 1, step);
          println!("     $ {}", cmd);
      }
  }
expected_output: |
  === プロジェクトセットアップチェックリスト ===

    1. Install Rust
       $ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    2. Install wasm-pack
       $ cargo install wasm-pack
    3. Create project
       $ cargo new --lib my-wasm-app
    4. Configure Cargo.toml
       $ crate-type = ["cdylib"]
    5. Build
       $ wasm-pack build --target web
---

## 前提条件

始める前に、以下のツールをインストールしてください：

```bash
# Rustのインストール（cargoを含む）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# wasm-packのインストール
cargo install wasm-pack

# インストール確認
rustc --version
wasm-pack --version
```

## プロジェクトの作成

```bash
# 新しいライブラリプロジェクトを作成
cargo new --lib my-wasm-app
cd my-wasm-app
```

## Cargo.tomlの設定

```toml
[package]
name = "my-wasm-app"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"

# 必要に応じて追加:
# web-sys = { version = "0.3", features = ["Window", "Document"] }
# js-sys = "0.3"
# wasm-bindgen-futures = "0.4"

[profile.release]
opt-level = "s"     # バイナリサイズの最適化
lto = true          # リンク時最適化
```

## 最初の関数を書く

`src/lib.rs`を以下の内容に置き換えます：

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

## ビルド

```bash
# <script type="module">で使用する場合のビルド
wasm-pack build --target web

# バンドラー（Vite、Webpack）向けのビルド
wasm-pack build --target bundler

# 最適化ビルド
wasm-pack build --target web --release
```

これにより、`.wasm`バイナリ、JavaScriptグルーコード、TypeScript型定義を含む`pkg/`ディレクトリが作成されます。

## HTMLでの使用

```html
<!DOCTYPE html>
<html>
<body>
  <script type="module">
    import init, { greet } from './pkg/my_wasm_app.js';

    async function run() {
      await init();
      document.body.textContent = greet("World");
    }
    run();
  </script>
</body>
</html>
```

## Viteでの使用

```bash
npm create vite@latest my-app -- --template vanilla-ts
cd my-app
npm install
```

```js
// src/main.ts
import init, { greet } from '../my-wasm-app/pkg';

async function run() {
  await init();
  console.log(greet("Vite"));
}
run();
```

## プロジェクト構成

```
my-wasm-app/
├── Cargo.toml          # Rustの依存関係
├── src/
│   └── lib.rs          # Rustコード
├── pkg/                # wasm-packが生成
│   ├── my_wasm_app_bg.wasm
│   ├── my_wasm_app.js
│   ├── my_wasm_app.d.ts
│   └── package.json
└── target/             # Rustビルド成果物
```

## バイナリサイズの最適化

```toml
# Cargo.toml
[profile.release]
opt-level = "s"       # "s" = 小さい、"z" = 最小
lto = true            # リンク時最適化
codegen-units = 1     # より良い最適化、コンパイルは遅くなる
strip = true          # デバッグシンボルを除去
```

さらにサイズを削減するために`wasm-opt`を実行します：

```bash
wasm-opt -Oz -o optimized.wasm pkg/my_wasm_app_bg.wasm
```

## 試してみよう

**Run**をクリックして、プロジェクトセットアップのチェックリストを確認しましょう。その後、自分のマシンで実際に試してみてください！
