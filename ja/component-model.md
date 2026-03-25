---
title: Wasmコンポーネントモデル
slug: component-model
difficulty: advanced
tags: [api]
order: 28
description: WebAssemblyの未来 — 合成可能なモジュール、型付きインターフェース、言語に依存しない連携可能なコンポーネント。
starter_code: |
  fn main() {
      println!("=== Wasm Component Model ===\n");

      println!("現在のWasm（コアモジュール）:");
      println!("  • 数値型のみ (i32, i64, f32, f64)");
      println!("  • 文字列はptr + lenで渡す（手動）");
      println!("  • 各モジュールは独立");
      println!("  • 言語固有のグルーコードが必要\n");

      println!("コンポーネントモデル（未来）:");
      println!("  • リッチな型: 文字列、リスト、レコード、バリアント");
      println!("  • 型付きインターフェース（WITファイル）");
      println!("  • モジュールがコンポーネントをインポート/エクスポート可能");
      println!("  • 言語非依存 — Rust、Go、Python、JS");
      println!("  • パッケージマネージャー（wargレジストリ）\n");

      println!("WIT（Wasm Interface Type）の例:");
      println!("  interface my-api {{");
      println!("    record user {{");
      println!("      name: string,");
      println!("      age: u32,");
      println!("    }}");
      println!("    greet: func(user: user) -> string");
      println!("  }}\n");

      println!("ツール:");
      println!("  • cargo-component — RustからWasmコンポーネントをビルド");
      println!("  • wasm-tools     — Wasmバイナリの操作");
      println!("  • wasmtime        — コンポーネント対応ランタイム");
      println!("  • jco             — コンポーネント用JSツール");
  }
expected_output: |
  === Wasm Component Model ===

  現在のWasm（コアモジュール）:
    • 数値型のみ (i32, i64, f32, f64)
    • 文字列はptr + lenで渡す（手動）
    • 各モジュールは独立
    • 言語固有のグルーコードが必要

  コンポーネントモデル（未来）:
    • リッチな型: 文字列、リスト、レコード、バリアント
    • 型付きインターフェース（WITファイル）
    • モジュールがコンポーネントをインポート/エクスポート可能
    • 言語非依存 — Rust、Go、Python、JS
    • パッケージマネージャー（wargレジストリ）

  WIT（Wasm Interface Type）の例:
    interface my-api {
      record user {
        name: string,
        age: u32,
      }
      greet: func(user: user) -> string
    }

  ツール:
    • cargo-component — RustからWasmコンポーネントをビルド
    • wasm-tools     — Wasmバイナリの操作
    • wasmtime        — コンポーネント対応ランタイム
    • jco             — コンポーネント用JSツール
---

## 現在のWasmの問題

今日のWebAssemblyは4つの型のみをサポートしています: `i32`、`i64`、`f32`、`f64`。それ以外（文字列、構造体、配列）は手動でエンコードする必要があります:

```
現在: Rust String → UTF-8バイト → ポインタ + 長さ → JSデコード
未来: Rust String → string（ネイティブWasm型）→ JS string
```

`wasm-bindgen` はRust↔JSではこの問題を解決しますが、**Rust固有**です。GoモジュールがカスタムグルーなしにRustモジュールの関数を呼ぶことはできません。

## コンポーネントモデルが解決するもの

| 問題 | 現在 | コンポーネントモデル |
|---------|---------|----------------|
| 型 | i32/i64/f32/f64のみ | 文字列、リスト、レコード、バリアント、オプション、リザルト |
| 相互運用 | 言語固有のグルー | 言語非依存のWITインターフェース |
| 合成 | モノリシックモジュール | インポート/エクスポート可能な合成可能コンポーネント |
| 依存関係 | 手動リンク | パッケージレジストリ（warg） |

## WIT: WebAssembly Interface Types

WITはコンポーネントのインターフェース定義言語です:

```wit
// my-api.wit
package example:my-api;

interface types {
    record user {
        name: string,
        age: u32,
        email: option<string>,
    }

    enum role {
        admin,
        editor,
        viewer,
    }

    variant error {
        not-found(string),
        permission-denied,
        internal(string),
    }
}

interface users {
    use types.{user, role, error};

    create-user: func(name: string, age: u32) -> result<user, error>;
    get-user: func(id: u32) -> option<user>;
    list-users: func() -> list<user>;
}

world my-app {
    import wasi:http/outgoing-handler;
    export users;
}
```

## WIT型

| WIT型 | 説明 | 例 |
|----------|------------|---------|
| `string` | UTF-8文字列 | `"hello"` |
| `bool` | ブーリアン | `true` |
| `u8`..`u64`, `s8`..`s64` | 整数 | `42` |
| `f32`, `f64` | 浮動小数点数 | `3.14` |
| `list<T>` | 動的配列 | `list<string>` |
| `option<T>` | null許容 | `option<u32>` |
| `result<T, E>` | 成功またはエラー | `result<user, error>` |
| `record` | 名前付きフィールド（構造体） | `record { name: string }` |
| `variant` | タグ付きユニオン（列挙型） | `variant { a(u32), b }` |
| `tuple<T, U>` | 匿名タプル | `tuple<string, u32>` |

## Rustでコンポーネントをビルドする

```bash
# cargo-componentをインストール
cargo install cargo-component

# 新しいコンポーネントプロジェクトを作成
cargo component new my-component
cd my-component
```

```rust
// src/lib.rs
#[allow(warnings)]
mod bindings;

use bindings::Guest;

struct Component;

impl Guest for Component {
    fn greet(name: String) -> String {
        format!("Hello, {}!", name)
    }
}

bindings::export!(Component with_types_in bindings);
```

```bash
# コンポーネントをビルド
cargo component build --release

# 出力は.wasmコンポーネント（コアモジュールではない）
```

## コンポーネントの合成

複数のコンポーネントを組み合わせることができます:

```
┌─────────────────┐     ┌─────────────────┐
│  認証コンポーネント│────▶│  APIコンポーネント │
│  (Rust)          │     │  (Go)            │
└─────────────────┘     └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │  DBコンポーネント  │
                        │  (Python)        │
                        └─────────────────┘
```

各コンポーネント:
- 型付きWITインターフェースを公開
- **任意の言語**で記述可能
- サンドボックス化 — インポートしたものだけにアクセス可能
- 他のコンポーネントを変更せずに置換可能

## タイムライン

| ステータス | 機能 |
|--------|---------|
| ✓ 安定版 | コアWasm（モジュール、メモリ、関数） |
| ✓ 安定版 | WASIプレビュー1（ファイルI/O、環境変数、引数） |
| ✓ 安定版 | コンポーネントモデル仕様 |
| ✓ 利用可能 | cargo-component、jco、wasmtimeサポート |
| 進行中 | WASIプレビュー2（HTTP、ソケット、キーバリュー） |
| 進行中 | wargパッケージレジストリ |
| 計画中 | ブラウザでのコンポーネントサポート |
| 計画中 | バンドラーでのコンポーネントリンク |

## なぜ重要なのか

コンポーネントモデルはWasmを「ブラウザ内の高速コード」から**ユニバーサルなソフトウェアコンポーネントフォーマット**へと変えます:

- Rustで画像処理を書く → Python、Go、JSから利用
- プラグインシステムを構築 → ユーザーは任意の言語でプラグインを記述
- マイクロサービス → 各サービスがサンドボックス化されたWasmコンポーネント
- エッジコンピューティング → CDNエッジでコンポーネントを合成

## 試してみよう

**Run** をクリックして、コンポーネントモデルの概要を確認しましょう — 提供される機能、WIT構文、利用可能なツールを確認できます。これがWebAssemblyの未来の方向性です。
