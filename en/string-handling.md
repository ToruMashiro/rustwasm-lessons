---
title: String Handling Deep Dive
slug: string-handling
difficulty: intermediate
tags: [data-structures]
order: 22
description: Master string passing between Rust and JavaScript — UTF-8 vs UTF-16 encoding, performance costs, and optimization strategies.
starter_code: |
  fn main() {
      // String encoding demonstration

      // 1. UTF-8 encoding (what Rust uses)
      let text = "Hello, 世界! 🌍";
      println!("Text: {}", text);
      println!("Byte length (UTF-8): {}", text.len());
      println!("Char count: {}", text.chars().count());

      // 2. Individual byte values
      let simple = "ABC";
      let bytes: Vec<u8> = simple.bytes().collect();
      println!("\n\"ABC\" as UTF-8 bytes: {:?}", bytes);

      // 3. Multi-byte characters
      let japanese = "日本語";
      println!("\n\"日本語\" UTF-8 bytes: {} bytes", japanese.len());
      println!("\"日本語\" chars: {} characters", japanese.chars().count());

      // 4. Emoji (4 bytes in UTF-8)
      let emoji = "🦀";
      println!("\n\"🦀\" UTF-8 bytes: {} bytes", emoji.len());

      // 5. String building (common in Wasm)
      let mut result = String::with_capacity(100);
      for i in 0..5 {
          result.push_str(&format!("item_{} ", i));
      }
      println!("\nBuilt string: {}", result.trim());
  }
expected_output: |
  Text: Hello, 世界! 🌍
  Byte length (UTF-8): 18
  Char count: 12

  "ABC" as UTF-8 bytes: [65, 66, 67]

  "日本語" UTF-8 bytes: 9 bytes
  "日本語" chars: 3 characters

  "🦀" UTF-8 bytes: 4 bytes

  Built string: item_0 item_1 item_2 item_3 item_4
---

## The Encoding Mismatch

This is the most common performance gotcha in Wasm:

| | Rust / Wasm | JavaScript |
|---|---|---|
| **Encoding** | UTF-8 | UTF-16 |
| **Bytes per ASCII char** | 1 | 2 |
| **Bytes per Japanese char** | 3 | 2 |
| **Bytes per emoji** | 4 | 4 |

Every time a string crosses the Wasm boundary, it must be **re-encoded**:

```
JS string (UTF-16) → encode to UTF-8 → copy into Wasm memory → Rust &str
Rust String (UTF-8) → decode from UTF-8 → create JS string (UTF-16)
```

This copy happens **every time** — there's no way to avoid it with current Wasm.

## How wasm-bindgen Handles Strings

### JS → Rust (`&str`)

```rust
#[wasm_bindgen]
pub fn process(input: &str) -> String {
    // input is already UTF-8 in Wasm memory
    // wasm-bindgen encoded it from JS UTF-16 before calling
    format!("Processed: {}", input)
    // Return String → wasm-bindgen decodes UTF-8 → JS gets UTF-16 string
}
```

Behind the scenes:
1. JS calls `TextEncoder.encode()` to convert UTF-16 → UTF-8
2. Copies UTF-8 bytes into Wasm linear memory
3. Passes pointer + length to Rust
4. Rust sees a valid `&str`

### Rust → JS (`String`)

1. Rust writes UTF-8 bytes to linear memory
2. wasm-bindgen calls `TextDecoder.decode()` to convert UTF-8 → UTF-16
3. JS gets a native string

## Performance Cost

```
Operation               Time (1MB string)
─────────────────────────────────────────
JS → Rust (&str)        ~2ms (encode + copy)
Rust → JS (String)      ~2ms (decode + copy)
Rust internal (no copy) ~0ms (just a pointer)
```

**Rule**: Minimize string boundary crossings. Do all string work on one side.

## Optimization Strategies

### 1. Batch operations — don't cross per-item

```rust
// BAD: crosses boundary N times
#[wasm_bindgen]
pub fn process_one(item: &str) -> String { /* ... */ }

// GOOD: crosses boundary once
#[wasm_bindgen]
pub fn process_all(items_json: &str) -> String {
    // Parse all items, process, return all results
    // Only 2 boundary crossings total
}
```

### 2. Use numeric IDs instead of strings

```rust
// BAD: passing strings back and forth
pub fn get_user_name(name: &str) -> String { /* ... */ }

// GOOD: pass ID, keep strings in Rust
pub fn create_user(name: &str) -> u32 { /* returns ID */ }
pub fn get_user_name(id: u32) -> String { /* lookup by ID */ }
```

### 3. Use &[u8] for binary data

```rust
// DON'T encode binary data as strings
pub fn process_base64(data: &str) -> String { /* ... */ }

// DO pass raw bytes
pub fn process_bytes(data: &[u8]) -> Vec<u8> { /* ... */ }
```

### 4. Pre-allocate with capacity

```rust
// BAD: many small allocations
let mut result = String::new();
for item in items {
    result.push_str(&format!("{},", item));
}

// GOOD: one allocation
let mut result = String::with_capacity(items.len() * 20);
for item in items {
    result.push_str(&format!("{},", item));
}
```

## Common String Patterns in Wasm

### CSV/JSON processing

```rust
#[wasm_bindgen]
pub fn parse_csv(input: &str) -> JsValue {
    let rows: Vec<Vec<&str>> = input
        .lines()
        .map(|line| line.split(',').collect())
        .collect();
    serde_wasm_bindgen::to_value(&rows).unwrap()
}
```

### Template rendering

```rust
#[wasm_bindgen]
pub fn render_template(template: &str, name: &str, count: u32) -> String {
    template
        .replace("{{name}}", name)
        .replace("{{count}}", &count.to_string())
}
```

## Try It

Click **Run** to see UTF-8 encoding in action — byte lengths, multi-byte characters, and string building. Notice how Japanese characters take 3 bytes in UTF-8 but only 2 in JavaScript's UTF-16.
