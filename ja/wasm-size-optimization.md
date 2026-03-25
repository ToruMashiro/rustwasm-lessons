---
title: Wasmバイナリサイズの最適化
slug: wasm-size-optimization
difficulty: intermediate
tags: [getting-started]
order: 34
description: Wasmバイナリを縮小するテクニックを学びます — Cargo.tomlの調整、wasm-opt、ツリーシェイキング、肥大化の回避、twiggyによる計測まで。
starter_code: |
  // サイズを意識したRustのパターンを示すデモ
  // （純粋なRust — コンセプトはWasmビルドに適用可能）

  use std::mem;

  // --- テクニック1: 動的型より固定サイズ型を使う ---
  struct CompactPoint {
      x: i16,
      y: i16,
  }

  struct HeavyPoint {
      x: f64,
      y: f64,
      label: String,  // Stringはアロケータ＋フォーマット機構を引き込む
  }

  // --- テクニック2: ホットパスでformat! / to_string()を避ける ---
  fn status_code_message(code: u16) -> &'static str {
      match code {
          200 => "OK",
          404 => "Not Found",
          500 => "Internal Server Error",
          _   => "Unknown",
      }
  }

  // --- テクニック3: 状態管理に文字列ではなくenumを使う ---
  #[repr(u8)]
  enum Color {
      Red   = 0,
      Green = 1,
      Blue  = 2,
  }

  // --- テクニック4: format!("{}", n)の代わりに手動itoaを使う ---
  fn u32_to_buf(mut n: u32, buf: &mut [u8; 10]) -> usize {
      if n == 0 {
          buf[0] = b'0';
          return 1;
      }
      let mut i = 0;
      while n > 0 {
          buf[i] = b'0' + (n % 10) as u8;
          n /= 10;
          i += 1;
      }
      buf[..i].reverse();
      i
  }

  fn main() {
      // サイズの比較
      println!("CompactPointのサイズ: {}バイト", mem::size_of::<CompactPoint>());
      println!("HeavyPoint のサイズ: {}バイト", mem::size_of::<HeavyPoint>());
      println!("Color enum のサイズ: {}バイト", mem::size_of::<Color>());

      // 静的文字列 — アロケータ不要
      println!("HTTP 200 -> {}", status_code_message(200));
      println!("HTTP 404 -> {}", status_code_message(404));

      // 手動の数値変換
      let mut buf = [0u8; 10];
      let len = u32_to_buf(42, &mut buf);
      let s = std::str::from_utf8(&buf[..len]).unwrap();
      println!("u32_to_buf(42) -> \"{}\"", s);

      // repr(u8) enumが1バイトであることを確認
      println!("Color::Red   as u8 = {}", Color::Red as u8);
      println!("Color::Green as u8 = {}", Color::Green as u8);
  }
expected_output: |
  CompactPointのサイズ: 4バイト
  HeavyPoint のサイズ: 40バイト
  Color enum のサイズ: 1バイト
  HTTP 200 -> OK
  HTTP 404 -> Not Found
  u32_to_buf(42) -> "42"
  Color::Red   as u8 = 0
  Color::Green as u8 = 1
---

## バイナリサイズが重要な理由

`.wasm`ファイルのすべてのバイトは、アプリケーションが起動する前にブラウザによってダウンロード、解析、コンパイルされる必要があります。JavaScriptとは異なり、Wasmはコンパクトなバイナリ形式ですが、素朴なRustビルドでは数メガバイトのファイルが生成されることもあります。

```
                    ダウンロード     解析          コンパイル
  ┌──────┐       ┌──────────┐    ┌─────────┐    ┌─────────┐
  │ .wasm│──────>│ ネットワーク │───>│ デコーダ │───>│ JIT/AOT │──> 実行
  │ file │       │ 転送     │    │         │    │ コンパイル│
  └──────┘       └──────────┘    └─────────┘    └─────────┘
     ^                ^
     │                │
   小さいファイル = どこでも高速な起動
```

| ファイルサイズ | 3G (2 Mbps) | 4G (20 Mbps) | Wi-Fi (50 Mbps) |
|-----------|-------------|--------------|-----------------|
| 100 KB    | 0.4 s       | 0.04 s       | 0.02 s          |
| 500 KB    | 2.0 s       | 0.2 s        | 0.08 s          |
| 2 MB      | 8.0 s       | 0.8 s        | 0.32 s          |
| 10 MB     | 40.0 s      | 4.0 s        | 1.6 s           |

## Cargo.tomlのリリースプロファイル

最も効果的な変更は、`[profile.release]`セクションの調整です：

```toml
[profile.release]
opt-level = "z"       # サイズを最適化 (s = サイズ、z = さらに小さく)
lto = true            # リンク時最適化 — プログラム全体の解析
codegen-units = 1     # 単一コード生成ユニット = インライン化/デッドコード除去の改善
strip = true          # デバッグシンボルの除去
panic = "abort"       # アンワインド機構なし（約10-20 KB節約）
```

### 各設定の効果

```
  opt-level = "z"
  ┌──────────────────────────────────────────────────────┐
  │ "0" — 最適化なし              （デバッグビルド）       │
  │ "1" — 基本的な最適化                                 │
  │ "2" — 標準的な最適化          （デフォルトリリース）   │
  │ "3" — 積極的な速度最適化                             │
  │ "s" — バイナリサイズを最適化                          │
  │ "z" — サイズをさらに強力に最適化                      │
  └──────────────────────────────────────────────────────┘

  lto = true
  ┌────────────────────────────────────────┐
  │ LTOなし:                              │
  │   crate_a.o ──┐                        │
  │   crate_b.o ──┼── リンク ── バイナリ   │
  │   crate_c.o ──┘  （デッドコードが残る） │
  │                                        │
  │ LTOあり:                              │
  │   crate_a.o ──┐                        │
  │   crate_b.o ──┼── 全体を ── 最適化     │
  │   crate_c.o ──┘  解析     ── バイナリ  │
  │                            （より小さい!）│
  └────────────────────────────────────────┘

  codegen-units = 1
  ┌─────────────────────────────────────────────────┐
  │ デフォルト: 16ユニット — コンパイルは速いが     │
  │ 最適化は少ない。                                │
  │ 1に設定: 単一ユニット — コンパイルは遅いが、    │
  │ コンパイラがすべてを把握し、インライン化と       │
  │ デッドコード除去がより効果的になる。             │
  └─────────────────────────────────────────────────┘
```

## wasm-opt: ビルド後オプティマイザ

Binaryenの`wasm-opt`は、Rustコンパイラでは行えないWasm固有の最適化を実行します：

```bash
# インストール
cargo install wasm-opt
# または: npm install -g binaryen

# サイズ最適化 (-Oz) または速度最適化 (-O3)
wasm-opt -Oz -o output.wasm input.wasm
```

`wasm-opt -Oz`による一般的な削減効果：

| 段階                | サイズ   |
|---------------------|----------|
| `cargo build`後     | 180 KB   |
| `wasm-opt`後        | 120 KB   |
| gzip後              | 45 KB    |
| brotli後            | 38 KB    |

## ツリーシェイキングとデッドコード除去

Rustの単相化（モノモーフィゼーション）は、ジェネリック関数の多くのコピーを生成する可能性があります。異なる`T`に対する`Vec<T>`の各インスタンス化が、個別のマシンコードを生成します。

```
  コードが使用:               単相化の結果:
  Vec<u8>          ──────> Vec_u8_push, Vec_u8_pop, Vec_u8_len ...
  Vec<String>      ──────> Vec_String_push, Vec_String_pop ...
  Vec<(i32,i32)>   ──────> Vec_tuple_push, Vec_tuple_pop ...
```

### 単相化による肥大化を減らす戦略：

1. **パフォーマンスが重要でない場合はトレイトオブジェクトを使用する**：
```rust
// 型ごとにコードを生成:
fn process<T: Display>(items: &[T]) { ... }

// 単一の関数を生成:
fn process(items: &[&dyn Display]) { ... }
```

2. **非ジェネリックコードをジェネリック関数から分離する**：
```rust
// 悪い例 — 関数全体がTごとに複製される
fn insert<T: Ord>(vec: &mut Vec<T>, item: T) {
    let idx = vec.binary_search(&item).unwrap_or_else(|i| i);
    vec.insert(idx, item);
    log_size(vec.len()); // この行はTを使わない
}

// 良い例 — log_sizeは一度だけコンパイルされる
fn log_size(n: usize) { /* ... */ }
fn insert<T: Ord>(vec: &mut Vec<T>, item: T) {
    let idx = vec.binary_search(&item).unwrap_or_else(|i| i);
    vec.insert(idx, item);
    log_size(vec.len());
}
```

## 隠れた肥大化の回避

### `format!`と`panic!`の問題

すべての`format!()`、`panic!()`、`unwrap()`、`expect()`呼び出しは、Rustのフォーマット機構を引き込みます — おおよそ**10〜20 KB**のコードです。

```rust
// 悪い例 — 範囲外アクセス時にDisplay + フォーマットをパニック文字列で引き込む
fn get(idx: usize) -> u8 {
    DATA[idx]  // 範囲外でフルフォーマットのパニック
}

// 良い例 — get() + 小さなabortを使用
fn get(idx: usize) -> u8 {
    match DATA.get(idx) {
        Some(&v) => v,
        None => abort_with_code(1),
    }
}
```

### 文字列肥大化チェックリスト

| パターン                       | バイナリに追加？        | 代替手段                     |
|--------------------------------|-------------------------|------------------------------|
| `format!("x = {}", x)`        | はい — フォーマット基盤 | 手動変換または &str           |
| `panic!("bad index {}", i)`    | はい — panic + format   | `unreachable_unchecked()`*   |
| `.unwrap()`                    | はい — パニック文字列   | `.unwrap_or(default)`        |
| `.expect("msg")`              | はい — panic + 文字列   | match + abort                |
| `#[derive(Debug)]` 型に対して  | はい — Debug実装        | 必要な場合のみderive          |
| 数値の`to_string()`           | はい — Displayトレイト  | itoaクレートまたは手動バッファ |

*`unreachable_unchecked`はunsafeであり、パスが到達不能であることを保証できる場合にのみ使用すべきです。

## wee_alloc: 小さなアロケータ

デフォルトのRustアロケータ（Wasmではdlmalloc）は約10 KBを追加します。`wee_alloc`はパフォーマンスとサイズをトレードオフします（約1 KB）：

```rust
// lib.rsに記述
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

```toml
[dependencies]
wee_alloc = "0.4"
```

トレードオフ：

| 特性              | dlmalloc     | wee_alloc    |
|-------------------|-------------|-------------|
| バイナリオーバーヘッド | ~10 KB  | ~1 KB       |
| アロケーション速度   | 高速       | 低速        |
| フラグメンテーション | 良好       | 不良        |
| 解放で回収？       | はい       | 部分的      |
| 最適な用途         | 汎用       | 小さなWasm  |

> **注意**: `wee_alloc`は現在メンテナンスされていません。本番環境では`lol_alloc`や`talc`クレート、または独自のバンプアロケータの実装を検討してください（レッスン35参照）。

## twiggyによる計測

`twiggy`は`.wasm`バイナリを解析し、何がスペースを占めているかを表示します：

```bash
# インストール
cargo install twiggy

# 上位20の最大アイテムを表示
twiggy top -n 20 my_app_bg.wasm

# 何が何を保持しているかのコールグラフを表示
twiggy dominators my_app_bg.wasm

# 2つのビルドを比較
twiggy diff old.wasm new.wasm
```

`twiggy top`の出力例：
```
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼────────────────────────────
         12480 │    8.31%  │ data[0]
          6204 │    4.13%  │ "function names" subsection
          3120 │    2.08%  │ core::fmt::write
          2800 │    1.86%  │ dlmalloc::dlmalloc::Dlmalloc::malloc
          2476 │    1.65%  │ core::fmt::Formatter::pad
```

## 圧縮: 最後のステップ

すべての最適化の後、HTTP圧縮を適用します。ほとんどのサーバーはgzipとBrotliをサポートしています：

```
  生の .wasm              gzip (-9)              Brotli (-11)
  ┌─────────┐            ┌─────────┐            ┌─────────┐
  │ 120 KB  │  ────────> │  45 KB  │  ────────> │  38 KB  │
  └─────────┘    ~62%    └─────────┘    ~15%    └─────────┘
                 削減                    さらに
```

Webサーバーの設定：
```nginx
# Nginx
location ~ \.wasm$ {
    gzip on;
    gzip_types application/wasm;
    # または事前圧縮ファイルを配信:
    gzip_static on;
    brotli_static on;
}
```

## 完全な最適化パイプライン

```
  cargo build --release     ← Cargo.tomlプロファイル設定
        │
        ▼
  wasm-opt -Oz              ← Binaryenによる後処理
        │
        ▼
  wasm-strip                ← namesセクションの除去（オプション）
        │
        ▼
  brotli / gzip             ← HTTP転送圧縮
        │
        ▼
  ✓ 最小限の .wasm をブラウザに配信
```

### クイックリファレンス: テクニック別の効果

| テクニック                       | 一般的な削減効果 |
|----------------------------------|-----------------|
| `opt-level = "z"`               | 10〜25%         |
| `lto = true`                    | 10〜20%         |
| `codegen-units = 1`             | 5〜10%          |
| `panic = "abort"`               | 5〜15%          |
| `strip = true`                  | 5〜15%          |
| `wasm-opt -Oz`                  | 15〜30%         |
| `wee_alloc`の置き換え            | 約10 KB固定     |
| `format!` / Debug deriveの除去   | 大きく変動      |
| Brotli圧縮                      | 60〜70%         |

## まとめ

バイナリサイズの最適化はスペクトラムです — すべてのプロジェクトにすべてのテクニックが必要なわけではありません。まずCargo.tomlの設定と`wasm-opt`（80/20の法則での勝利）から始め、次に`twiggy`を使って残りの肥大化を見つけましょう。50 KB未満を目指す場合は、フォーマット基盤の回避とカスタムアロケータの検討が必要になります。
