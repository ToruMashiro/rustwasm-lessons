---
title: Testing Wasm
slug: testing-wasm
difficulty: intermediate
tags: [getting-started]
order: 18
description: Test Rust/Wasm code with wasm-pack test, headless browsers, and unit tests that run both natively and in the browser.
starter_code: |
  // Unit tests run with: cargo test
  // Wasm tests run with: wasm-pack test --headless --chrome

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
      // Running tests manually
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

      println!("\nAll tests passed!");
  }
expected_output: |
  ✓ add(2, 3) == 5
  ✓ add(-1, 1) == 0
  ✓ fibonacci(0) == 0
  ✓ fibonacci(10) == 55
  ✓ fibonacci(20) == 6765

  All tests passed!
---

## Why Test Wasm?

Rust/Wasm code runs in two environments — native (your machine) and browser (Wasm). Tests should cover both to catch environment-specific bugs.

## Test Setup

```toml
# Cargo.toml
[dev-dependencies]
wasm-bindgen-test = "0.3"
```

## Unit Tests (Native)

Standard Rust tests run with `cargo test`:

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

## Wasm Tests (Browser)

Tests that need browser APIs use `wasm-bindgen-test`:

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
# Run in headless Chrome
wasm-pack test --headless --chrome

# Run in headless Firefox
wasm-pack test --headless --firefox

# Run in Node.js (no DOM available)
wasm-pack test --node
```

## Async Tests

Test async Wasm functions:

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

## Test Organization

```
src/
├── lib.rs          # Your Wasm code
└── tests/          # Integration tests
    ├── native.rs   # cargo test (no browser)
    └── web.rs      # wasm-pack test (needs browser)
```

## Testing Best Practices

| Practice | Why |
|----------|-----|
| Test pure logic natively (`cargo test`) | Fast, no browser needed |
| Test DOM/browser interactions with `wasm-bindgen-test` | Catches Wasm-specific issues |
| Use `--headless` in CI | No GUI needed on CI servers |
| Test error cases | `Result::Err` paths are often untested |
| Benchmark with `console.time()` | Catch performance regressions |

## CI Configuration (GitHub Actions)

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

## Try It

Click **Run** to see the test assertions pass. In real projects, these would be in `#[test]` and `#[wasm_bindgen_test]` functions.
