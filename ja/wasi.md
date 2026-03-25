---
title: ブラウザ外のWasm（WASI）
slug: wasi
difficulty: advanced
tags: [api]
order: 25
description: WASIでWebAssemblyをブラウザ外で実行する — サーバーサイドWasm、エッジコンピューティング、ポータブルバイナリ。
starter_code: |
  fn main() {
      println!("=== WASI (WebAssembly System Interface) ===\n");

      println!("WASIが提供するもの:");
      let capabilities = [
          ("File I/O", "ケーパビリティベースセキュリティによるファイル読み書き"),
          ("Env vars", "環境変数へのアクセス"),
          ("Args", "コマンドライン引数の読み取り"),
          ("Clocks", "壁時計とモノトニックタイマー"),
          ("Random", "暗号学的乱数"),
          ("Stdout/Stderr", "標準出力ストリーム"),
      ];

      for (cap, desc) in &capabilities {
          println!("  • {}: {}", cap, desc);
      }

      println!("\nWASIをサポートするランタイム:");
      let runtimes = ["Wasmtime", "Wasmer", "WasmEdge", "Spin", "Cloudflare Workers"];
      for rt in &runtimes {
          println!("  - {}", rt);
      }

      println!("\nWASI向けにコンパイル:");
      println!("  rustup target add wasm32-wasip1");
      println!("  cargo build --target wasm32-wasip1 --release");
      println!("  wasmtime ./target/wasm32-wasip1/release/my_app.wasm");
  }
expected_output: |
  === WASI (WebAssembly System Interface) ===

  WASIが提供するもの:
    • File I/O: ケーパビリティベースセキュリティによるファイル読み書き
    • Env vars: 環境変数へのアクセス
    • Args: コマンドライン引数の読み取り
    • Clocks: 壁時計とモノトニックタイマー
    • Random: 暗号学的乱数
    • Stdout/Stderr: 標準出力ストリーム

  WASIをサポートするランタイム:
    - Wasmtime
    - Wasmer
    - WasmEdge
    - Spin
    - Cloudflare Workers

  WASI向けにコンパイル:
    rustup target add wasm32-wasip1
    cargo build --target wasm32-wasip1 --release
    wasmtime ./target/wasm32-wasip1/release/my_app.wasm
---

## WASIとは？

WASI（WebAssembly System Interface）は、Wasmを**ブラウザ外**で実行できるようにします — サーバー、エッジネットワーク、IoTデバイス、CLIツールで動作します。ブラウザが必要としないシステム機能（ファイルI/O、ネットワーキングなど）への標準化されたAPIを提供します。

```
ブラウザWasm                     WASI Wasm
┌───────────────┐               ┌───────────────┐
│ Web API       │               │ システムAPI    │
│ (DOM, Canvas, │               │ (ファイル, Net,│
│  Fetch等)     │               │  Env, Clock)  │
├───────────────┤               ├───────────────┤
│ Wasmランタイム │               │ Wasmランタイム │
│ (V8, Spider-  │               │ (Wasmtime,    │
│  Monkey)      │               │  Wasmer)      │
└───────────────┘               └───────────────┘
```

## 2つのWasmターゲット

| ターゲット | 実行場所 | API |
|--------|--------------|------|
| `wasm32-unknown-unknown` | ブラウザ | web-sys経由のWeb API |
| `wasm32-wasip1` | サーバー/CLI | WASI（ファイル、環境変数、引数） |

## はじめ方

```bash
# WASIターゲットを追加
rustup target add wasm32-wasip1

# WASIランタイムをインストール
cargo install wasmtime-cli

# プロジェクトを作成
cargo new --bin my-wasi-app
cd my-wasi-app
```

```rust
// src/main.rs — ネイティブRustと同様にWASI上で動作
use std::fs;
use std::env;

fn main() {
    // コマンドライン引数
    let args: Vec<String> = env::args().collect();
    println!("Args: {:?}", args);

    // ファイルI/O（サンドボックス内）
    fs::write("output.txt", "Hello from WASI!").unwrap();
    let content = fs::read_to_string("output.txt").unwrap();
    println!("Read: {}", content);

    // 環境変数
    if let Ok(val) = env::var("MY_VAR") {
        println!("MY_VAR = {}", val);
    }
}
```

```bash
# ビルド
cargo build --target wasm32-wasip1 --release

# wasmtimeで実行（ファイルアクセスを許可）
wasmtime --dir=. ./target/wasm32-wasip1/release/my-wasi-app.wasm
```

## ケーパビリティベースセキュリティ

WASIは**サンドボックスモデル**を採用しています — Wasmモジュールはデフォルトでアクセス権がありません。ケーパビリティを明示的に付与します:

```bash
# 何にもアクセスできない
wasmtime app.wasm

# カレントディレクトリへのアクセスを許可
wasmtime --dir=. app.wasm

# 特定のパスへのアクセスを許可
wasmtime --dir=/data::/app/data app.wasm

# 環境変数を許可
wasmtime --env MY_VAR=hello app.wasm
```

これはネイティブ実行ファイルよりも根本的に安全です — Wasmモジュールは明示的に許可しない限り、ファイル、ネットワーク、環境変数を読み取れません。

## ユースケース

| ユースケース | WASIを使う理由 |
|----------|----------|
| **エッジコンピューティング**（Cloudflare Workers） | コールドスタート ~1ms（コンテナは~100ms） |
| **プラグインシステム** | アプリ内で信頼できないコードを安全に実行 |
| **CLIツール** | 一度書けばWasmランタイムのある任意のプラットフォームで実行 |
| **サーバーレス関数** | コンテナより小さく、速く、安全 |
| **組み込み/IoT** | 極小のランタイムフットプリント（約5MB） |

## WASI vs ブラウザWasm

| 機能 | ブラウザ | WASI |
|---------|---------|------|
| DOMアクセス | ✓ | ✗ |
| ファイルシステム | ✗ | ✓（サンドボックス内） |
| ネットワークソケット | ✗（Fetchのみ） | ✓（サンドボックス内） |
| 起動時間 | ~100ms | ~1ms |
| サンドボックス | ブラウザサンドボックス | ケーパビリティベース |

## Cloudflare Workersの例

Cloudflare WorkersはWASI Wasmを実行できます:

```rust
// WASIにコンパイルされるシンプルなHTTPハンドラー
fn main() {
    let body = "Hello from Rust + WASI on Cloudflare!";
    println!("Content-Type: text/plain");
    println!("Content-Length: {}", body.len());
    println!();
    print!("{}", body);
}
```

## 未来: コンポーネントモデル

WASIは**コンポーネントモデル**に向けて進化しています — 型付きインターフェースをインポート/エクスポートできる、合成可能なWasmモジュールです（バイトだけでなく）。これにより以下が可能になります:
- Wasmモジュール同士の呼び出し
- 言語に依存しないインターフェース
- Wasmコンポーネント用のパッケージマネージャー

## 試してみよう

**Run** をクリックして、WASIのケーパビリティを確認しましょう。コードはWASIが提供するものとコンパイル方法を一覧表示します。ローカルで `wasmtime` を使って実行し、ファイルI/Oや環境変数の動作を確認してみてください。
