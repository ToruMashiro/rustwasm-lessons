---
title: Wasmにおけるエラーハンドリング
slug: error-handling
difficulty: intermediate
tags: [getting-started]
order: 17
description: Result、JsValue、カスタムエラー、パニックフックを使用して、Rust/JavaScript境界でエラーを適切に処理する方法を学びます。
starter_code: |
  fn main() {
      // Wasmのエラーハンドリングパターンをシミュレーション

      // パターン1: 文字列エラーを持つResult
      let result = divide(10.0, 0.0);
      match result {
          Ok(v) => println!("10 / 0 = {}", v),
          Err(e) => println!("Error: {}", e),
      }

      // パターン2: OptionからResultへの変換
      let item = find_item(99);
      match item {
          Ok(name) => println!("Found: {}", name),
          Err(e) => println!("Error: {}", e),
      }

      // パターン3: ?演算子によるチェーン
      match process_data("42") {
          Ok(v) => println!("Processed: {}", v),
          Err(e) => println!("Error: {}", e),
      }
      match process_data("abc") {
          Ok(v) => println!("Processed: {}", v),
          Err(e) => println!("Error: {}", e),
      }
  }

  fn divide(a: f64, b: f64) -> Result<f64, String> {
      if b == 0.0 {
          Err("Division by zero".to_string())
      } else {
          Ok(a / b)
      }
  }

  fn find_item(id: u32) -> Result<String, String> {
      let items = vec!["apple", "banana", "cherry"];
      items.get(id as usize)
          .map(|s| s.to_string())
          .ok_or(format!("Item {} not found", id))
  }

  fn process_data(input: &str) -> Result<i32, String> {
      let num: i32 = input.parse().map_err(|e: std::num::ParseIntError| e.to_string())?;
      Ok(num * 2)
  }
expected_output: |
  Error: Division by zero
  Error: Item 99 not found
  Processed: 84
  Error: invalid digit found in string
---

## Wasmでエラーハンドリングが重要な理由

RustがWasm内でパニックすると、ブラウザには不可解な `unreachable` エラーが表示されます。適切なエラーハンドリングにより、RustのエラーをJavaScriptの意味のある例外に変換できます。

## 2つのアプローチ

### 1. `Result<T, JsValue>` — エラーがJS例外になる

```rust
#[wasm_bindgen]
pub fn safe_divide(a: f64, b: f64) -> Result<f64, JsValue> {
    if b == 0.0 {
        Err(JsValue::from_str("Division by zero"))
    } else {
        Ok(a / b)
    }
}
```

JavaScript側：
```js
try {
    const result = safe_divide(10, 0);
} catch (e) {
    console.error(e); // "Division by zero"
}
```

### 2. `Option<T>` — 失敗時にundefinedを返す

```rust
#[wasm_bindgen]
pub fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".to_string()) } else { None }
}
```

JavaScript側：
```js
const user = find_user(99);
if (user === undefined) {
    console.log("User not found");
}
```

## エラー型の変換

`?` 演算子は `Result` と共に動作しますが、Wasmでは `JsValue` エラーが必要です：

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_and_double(input: &str) -> Result<i32, JsValue> {
    let num: i32 = input.parse()
        .map_err(|e: std::num::ParseIntError| {
            JsValue::from_str(&format!("Parse error: {}", e))
        })?;
    Ok(num * 2)
}
```

## カスタムエラー型

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct WasmError {
    code: u32,
    message: String,
}

#[wasm_bindgen]
impl WasmError {
    pub fn code(&self) -> u32 { self.code }
    pub fn message(&self) -> String { self.message.clone() }
}

fn make_error(code: u32, msg: &str) -> JsValue {
    let err = WasmError { code, message: msg.to_string() };
    JsValue::from(err)
}
```

## パニックフック — 最後の手段

防ぎきれないパニックに対しては、パニックフックをインストールすることで読みやすいメッセージを取得できます：

```rust
use console_error_panic_hook;

#[wasm_bindgen(start)]
pub fn init() {
    console_error_panic_hook::set_once();
}
```

フックなし: `RuntimeError: unreachable`
フックあり: `panicked at 'index out of bounds: the len is 3 but the index is 5'`

## ベストプラクティス

| すべきこと | 避けるべきこと |
|---|---|
| エクスポートされた関数から `Result<T, JsValue>` を返す | 関数をパニックさせる |
| `?` と `.map_err()` でエラーを変換する | エクスポートされた関数で `.unwrap()` を使う |
| `console_error_panic_hook` をインストールする | パニックを無視する |
| 意味のあるエラーメッセージを提供する | 「エラーが発生しました」のような汎用メッセージを返す |
| 「見つからない」ケースに `Option<T>` を使う | センチネル値（-1、空文字列）を返す |

## 試してみよう

**Run** をクリックして、エラーハンドリングパターンの動作を確認してください。各エラー型がどのように明確で説明的なメッセージを生成するかに注目してください。
