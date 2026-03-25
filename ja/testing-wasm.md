---
title: Wasmのテスト
slug: testing-wasm
difficulty: intermediate
tags: [getting-started]
order: 18
description: wasm-pack test、ヘッドレスブラウザ、ネイティブとブラウザの両方で動作するユニットテストを使ってRust/Wasmコードをテストする方法を学びます。
starter_code: |
  // ユニットテストの実行: cargo test
  // Wasmテストの実行: wasm-pack test --headless --chrome

  fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  fn fibonacci(n: u32) -> u32 {
      match n {
          0 => 0,
          1 => 1,
          _ => {
              let mut a = 0u32;
              let mut b = 1u32;
              for _ in 2..=n {
                  let temp = b;
                  b = a + b;
                  a = temp;
              }
              b
          }
      }
  }

  fn main() {
      // テストを手動で実行
      assert_eq!(add(2, 3), 5);
      println!("✓ add(2, 3) == 5");

      assert_eq!(add(-1, 1), 0);
      println!("✓ add(-1, 1) == 0");

      assert_eq!(fibonacci(0), 0);
      println!("✓ fibonacci(0) == 0");

      assert_eq!(fibonacci(10), 55);
      println!("✓ fibonacci(10) == 55");

      assert_eq!(fibonacci(20), 6765);
      println!("✓ fibonacci(20) == 6765");

      println!("\n全てのテストが成功しました！");
  }
expected_output: |
  ✓ add(2, 3) == 5
  ✓ add(-1, 1) == 0
  ✓ fibonacci(0) == 0
  ✓ fibonacci(10) == 55
  ✓ fibonacci(20) == 6765

  全てのテストが成功しました！
---

## なぜWasmをテストするのか？

Rust/Wasmコードはネイティブ（ローカルマシン）とブラウザ（Wasm）の2つの環境で動作します。環境固有のバグを検出するために、両方の環境でテストする必要があります。

## テストのセットアップ

```toml
# Cargo.toml
[dev-dependencies]
wasm-bindgen-test = "0.3"
```

## ユニットテスト（ネイティブ）

標準のRustテストは `cargo test` で実行します：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_eq!(add(-1, 1), 0);
        assert_eq!(add(0, 0), 0);
    }

    #[test]
    fn test_fibonacci() {
        assert_eq!(fibonacci(0), 0);
        assert_eq!(fibonacci(1), 1);
        assert_eq!(fibonacci(10), 55);
    }
}
```

```bash
cargo test
```

## Wasmテスト（ブラウザ）

ブラウザAPIを必要とするテストには `wasm-bindgen-test` を使用します：

```rust
#[cfg(test)]
mod wasm_tests {
    use wasm_bindgen_test::*;
    use super::*;

    wasm_bindgen_test_configure!(run_in_browser);

    #[wasm_bindgen_test]
    fn test_greet() {
        let result = greet("World");
        assert_eq!(result, "Hello, World!");
    }

    #[wasm_bindgen_test]
    fn test_dom_creation() {
        let document = web_sys::window().unwrap().document().unwrap();
        let el = document.create_element("div").unwrap();
        el.set_text_content(Some("test"));
        assert_eq!(el.text_content().unwrap(), "test");
    }
}
```

```bash
# ヘッドレスChromeで実行
wasm-pack test --headless --chrome

# ヘッドレスFirefoxで実行
wasm-pack test --headless --firefox

# Node.jsで実行（DOMは使用不可）
wasm-pack test --node
```

## 非同期テスト

非同期Wasm関数のテスト：

```rust
#[wasm_bindgen_test]
async fn test_async_fetch() {
    let window = web_sys::window().unwrap();
    let resp = JsFuture::from(
        window.fetch_with_str("/test-data.json")
    ).await.unwrap();
    assert!(resp.is_object());
}
```

## テストの構成

```
src/
├── lib.rs          # Wasmコード
└── tests/          # 統合テスト
    ├── native.rs   # cargo test（ブラウザ不要）
    └── web.rs      # wasm-pack test（ブラウザ必要）
```

## テストのベストプラクティス

| プラクティス | 理由 |
|----------|-----|
| 純粋なロジックはネイティブでテスト（`cargo test`） | 高速でブラウザ不要 |
| DOM/ブラウザ操作は `wasm-bindgen-test` でテスト | Wasm固有の問題を検出できる |
| CIでは `--headless` を使用 | CIサーバーにGUIは不要 |
| エラーケースもテスト | `Result::Err` のパスはテストされないことが多い |
| `console.time()` でベンチマーク | パフォーマンスの低下を検出できる |

## CI設定（GitHub Actions）

```yaml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: cargo-bins/cargo-binstall@main
      - run: cargo binstall wasm-pack -y
      - run: cargo test
      - run: wasm-pack test --headless --chrome
```

## 試してみよう

**Run** をクリックして、テストのアサーションが成功するのを確認してください。実際のプロジェクトでは、これらは `#[test]` や `#[wasm_bindgen_test]` 関数内に記述します。
