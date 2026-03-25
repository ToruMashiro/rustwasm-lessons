---
title: Error Handling in Wasm
slug: error-handling
difficulty: intermediate
tags: [getting-started]
order: 17
description: Handle errors gracefully across the Rust/JavaScript boundary using Result, JsValue, custom errors, and panic hooks.
starter_code: |
  fn main() {
      // Simulating Wasm error handling patterns

      // Pattern 1: Result with string error
      let result = divide(10.0, 0.0);
      match result {
          Ok(v) => println!("10 / 0 = {}", v),
          Err(e) => println!("Error: {}", e),
      }

      // Pattern 2: Option to Result conversion
      let item = find_item(99);
      match item {
          Ok(name) => println!("Found: {}", name),
          Err(e) => println!("Error: {}", e),
      }

      // Pattern 3: Chaining with ?
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

## Why Error Handling Matters in Wasm

When Rust panics in Wasm, the browser shows a cryptic `unreachable` error. Proper error handling converts Rust errors into meaningful JavaScript exceptions.

## The Two Approaches

### 1. `Result<T, JsValue>` — Errors become JS exceptions

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

In JavaScript:
```js
try {
    const result = safe_divide(10, 0);
} catch (e) {
    console.error(e); // "Division by zero"
}
```

### 2. `Option<T>` — Returns undefined on failure

```rust
#[wasm_bindgen]
pub fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".to_string()) } else { None }
}
```

In JavaScript:
```js
const user = find_user(99);
if (user === undefined) {
    console.log("User not found");
}
```

## Converting Between Error Types

The `?` operator works with `Result`, but Wasm needs `JsValue` errors:

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

## Custom Error Types

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

## Panic Hook — Last Resort

For panics you can't prevent, install the panic hook to get readable messages:

```rust
use console_error_panic_hook;

#[wasm_bindgen(start)]
pub fn init() {
    console_error_panic_hook::set_once();
}
```

Without it: `RuntimeError: unreachable`
With it: `panicked at 'index out of bounds: the len is 3 but the index is 5'`

## Best Practices

| Do | Don't |
|---|---|
| Return `Result<T, JsValue>` from exported functions | Let functions panic |
| Use `?` with `.map_err()` to convert errors | Use `.unwrap()` in exported functions |
| Install `console_error_panic_hook` | Ignore panics |
| Give meaningful error messages | Return generic "error occurred" |
| Use `Option<T>` for "not found" cases | Return sentinel values (-1, empty string) |

## Try It

Click **Run** to see the error handling patterns in action. Notice how each error type produces a clear, descriptive message.
