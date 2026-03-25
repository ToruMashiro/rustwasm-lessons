---
title: Debugging Wasm
slug: debugging
difficulty: beginner
tags: [getting-started]
order: 10
description: Debug Rust/Wasm applications using console logging, panic hooks, browser DevTools, and error handling best practices.
starter_code: |
  fn main() {
      // Simulating common debug scenarios

      // 1. Basic logging
      println!("Debug: starting application");

      // 2. Variable inspection
      let data = vec![1, 2, 3, 4, 5];
      println!("Debug: data = {:?}", data);
      println!("Debug: data length = {}", data.len());

      // 3. Conditional debugging
      let value = 42;
      if value > 40 {
          println!("Warning: value {} exceeds threshold", value);
      }

      // 4. Error simulation
      let result: Result<i32, String> = Err("something went wrong".to_string());
      match result {
          Ok(v) => println!("Success: {}", v),
          Err(e) => println!("Error: {}", e),
      }

      println!("\nAll debug checks passed!");
  }
expected_output: |
  Debug: starting application
  Debug: data = [1, 2, 3, 4, 5]
  Debug: data length = 5
  Warning: value 42 exceeds threshold
  Error: something went wrong

  All debug checks passed!
---

## Why Debugging Wasm is Different

When Rust panics in Wasm, you get an unhelpful `unreachable` error in the browser console. No stack trace, no error message. This lesson shows you how to fix that.

## Step 1: Better Panic Messages

Add `console_error_panic_hook` to get readable panic messages:

```toml
[dependencies]
console_error_panic_hook = "0.1"
wasm-bindgen = "0.2"
```

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn init() {
    // Call this once at startup
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn might_panic(x: i32) -> i32 {
    // Without the hook: "unreachable"
    // With the hook: "panicked at 'index out of bounds: len is 3, index is 5'"
    let v = vec![1, 2, 3];
    v[x as usize]
}
```

## Step 2: Console Logging from Rust

### Simple: Use `web_sys::console`

```rust
use web_sys::console;

// Log a string
console::log_1(&"Hello from Rust!".into());

// Log with formatting
console::log_2(&"Value:".into(), &JsValue::from(42));

// Warn and error
console::warn_1(&"This is a warning".into());
console::error_1(&"This is an error".into());
```

### Cleaner: Create a log macro

```rust
macro_rules! log {
    ($($t:tt)*) => {
        web_sys::console::log_1(
            &wasm_bindgen::JsValue::from_str(&format!($($t)*))
        );
    };
}

// Usage:
log!("User {} logged in at {}", name, timestamp);
log!("Data: {:?}", my_vec);
```

## Step 3: Browser DevTools

### Source Maps

Build with debug info to see Rust source in DevTools:

```bash
# Debug build (larger binary, has debug info)
wasm-pack build --dev --target web

# Release build (no debug info)
wasm-pack build --target web
```

### Network Tab

Check that your `.wasm` file is loading correctly:
- Status should be 200
- Content-Type should be `application/wasm`
- Size tells you if optimization is working

### Performance Tab

Profile Wasm execution:
1. Open DevTools → Performance
2. Click Record
3. Interact with your app
4. Stop recording
5. Look for "wasm-function" entries in the flame chart

## Step 4: Error Handling Patterns

### Return Result instead of panicking

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

In JavaScript, this becomes a try/catch:

```js
try {
    const result = safe_divide(10, 0);
} catch (e) {
    console.error("Wasm error:", e); // "Division by zero"
}
```

### Custom error types

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct AppError {
    message: String,
    code: u32,
}

#[wasm_bindgen]
impl AppError {
    pub fn message(&self) -> String { self.message.clone() }
    pub fn code(&self) -> u32 { self.code }
}
```

## Common Debugging Checklist

| Problem | Cause | Fix |
|---------|-------|-----|
| `unreachable` in console | Rust panic without hook | Add `console_error_panic_hook` |
| `.wasm` file not loading | Wrong path or MIME type | Check Network tab, serve with correct Content-Type |
| `LinkError` on load | Missing imports | Check that all `extern "C"` imports are satisfied |
| `RuntimeError: memory` | Out of Wasm memory | Increase memory or fix memory leak |
| Function not found | Not exported | Add `#[wasm_bindgen]` and `pub` |
| Types don't match | Wrong wasm-bindgen version | Ensure JS glue and .wasm are from same build |

## Try It

Click **Run** to see the debug output examples. In real Wasm projects, replace `println!` with `console::log_1`.
