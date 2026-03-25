---
title: Serde Serialization for Wasm
slug: serde-wasm
difficulty: intermediate
tags: [data-structures]
order: 37
description: Master serde serialization in Wasm — compare serde-wasm-bindgen vs JsValue, serialize complex types, and optimize for performance.
starter_code: |
  // Demonstrating serde serialization concepts in plain Rust
  // using serde_json (the same patterns apply to serde-wasm-bindgen)

  use std::collections::HashMap;
  use std::fmt;

  // --- Simulating serde without the crate ---
  // In real code you'd #[derive(Serialize, Deserialize)]

  #[derive(Debug, Clone)]
  struct User {
      name: String,
      age: u32,
      email: Option<String>,
      role: Role,
  }

  #[derive(Debug, Clone)]
  enum Role {
      Admin,
      Editor,
      Viewer,
  }

  #[derive(Debug, Clone)]
  struct ApiResponse {
      status: u16,
      data: Vec<User>,
      metadata: HashMap<String, String>,
  }

  // Manual JSON serialization (serde does this automatically)
  impl fmt::Display for Role {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          match self {
              Role::Admin  => write!(f, "\"admin\""),
              Role::Editor => write!(f, "\"editor\""),
              Role::Viewer => write!(f, "\"viewer\""),
          }
      }
  }

  impl User {
      fn to_json(&self) -> String {
          let email = match &self.email {
              Some(e) => format!("\"{}\"", e),
              None => "null".to_string(),
          };
          format!(
              r#"{{"name":"{}","age":{},"email":{},"role":{}}}"#,
              self.name, self.age, email, self.role
          )
      }

      fn from_fields(name: &str, age: u32, email: Option<&str>, role: Role) -> Self {
          User {
              name: name.to_string(),
              age,
              email: email.map(|e| e.to_string()),
              role,
          }
      }
  }

  impl ApiResponse {
      fn to_json(&self) -> String {
          let users: Vec<String> = self.data.iter().map(|u| u.to_json()).collect();
          let meta: Vec<String> = self.metadata.iter()
              .map(|(k, v)| format!(r#""{}":"{}""#, k, v))
              .collect();
          format!(
              r#"{{"status":{},"data":[{}],"metadata":{{{}}}}}"#,
              self.status,
              users.join(","),
              meta.join(",")
          )
      }
  }

  fn main() {
      println!("=== Serde Serialization Demo ===\n");

      // Create test data
      let alice = User::from_fields("Alice", 30, Some("alice@example.com"), Role::Admin);
      let bob = User::from_fields("Bob", 25, None, Role::Viewer);

      // Serialize individual users
      println!("Alice JSON:");
      println!("  {}", alice.to_json());
      println!();
      println!("Bob JSON:");
      println!("  {}", bob.to_json());
      println!();

      // Serialize a complex nested response
      let mut metadata = HashMap::new();
      metadata.insert("page".to_string(), "1".to_string());
      metadata.insert("total".to_string(), "2".to_string());

      let response = ApiResponse {
          status: 200,
          data: vec![alice.clone(), bob.clone()],
          metadata,
      };

      println!("API Response JSON:");
      println!("  {}", response.to_json());
      println!();

      // Demonstrate Option handling
      println!("Option<String> serialization:");
      println!("  Some(email) -> {}", format!(r#""{}""#, alice.email.as_ref().unwrap()));
      println!("  None        -> null");
      println!();

      // Demonstrate enum serialization
      println!("Enum variants:");
      println!("  Admin  -> {}", Role::Admin);
      println!("  Editor -> {}", Role::Editor);
      println!("  Viewer -> {}", Role::Viewer);
  }
expected_output: |
  === Serde Serialization Demo ===

  Alice JSON:
    {"name":"Alice","age":30,"email":"alice@example.com","role":"admin"}

  Bob JSON:
    {"name":"Bob","age":25,"email":null,"role":"viewer"}

  API Response JSON:
    {"status":200,"data":[{"name":"Alice","age":30,"email":"alice@example.com","role":"admin"},{"name":"Bob","age":25,"email":null,"role":"viewer"}],"metadata":{"page":"1","total":"2"}}

  Option<String> serialization:
    Some(email) -> "alice@example.com"
    None        -> null

  Enum variants:
    Admin  -> "admin"
    Editor -> "editor"
    Viewer -> "viewer"
---

## Why Serde Matters in Wasm

When Rust/Wasm communicates with JavaScript, data must cross the **Wasm boundary**. Rust structs live in linear memory; JavaScript objects live on the JS heap. Serde bridges this gap by converting between the two representations.

```
  Rust/Wasm side                JS side
  ┌────────────────┐          ┌────────────────┐
  │ struct User {  │          │ { name: "Ali", │
  │   name: String │  serde   │   age: 30,     │
  │   age: u32     │ <======> │   role: "admin"}│
  │   role: Role   │          │                │
  │ }              │          │ (JS Object)    │
  └────────────────┘          └────────────────┘
       Linear Memory               JS Heap
```

## Two Approaches: serde-wasm-bindgen vs serde_json

### Approach 1: serde-wasm-bindgen (Recommended)

Converts directly between Rust types and `JsValue` without going through a JSON string:

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
struct Point { x: f64, y: f64 }

#[wasm_bindgen]
pub fn process_point(val: JsValue) -> Result<JsValue, JsValue> {
    let point: Point = serde_wasm_bindgen::from_value(val)?;
    let result = Point { x: point.x * 2.0, y: point.y * 2.0 };
    Ok(serde_wasm_bindgen::to_value(&result)?)
}
```

### Approach 2: serde_json (via JsValue string)

Serializes to a JSON string, then parses on the other side:

```rust
#[wasm_bindgen]
pub fn process_point_json(json: &str) -> String {
    let point: Point = serde_json::from_str(json).unwrap();
    let result = Point { x: point.x * 2.0, y: point.y * 2.0 };
    serde_json::to_string(&result).unwrap()
}
```

### Comparison

```
  serde-wasm-bindgen path:
  Rust struct ──> JsValue  (direct, one step)

  serde_json path:
  Rust struct ──> JSON String ──> JS JSON.parse() ──> JS Object
                     ^                ^
                 serialize          parse
                 (Rust)           (JS engine)
```

| Feature                     | serde-wasm-bindgen    | serde_json              |
|-----------------------------|-----------------------|-------------------------|
| Intermediate format         | None (direct)         | JSON string             |
| Supports Map, Set, Date     | Yes (native JS types) | No (JSON subset only)   |
| Supports BigInt             | Yes                   | No                      |
| Supports undefined vs null  | Yes                   | No (both become null)   |
| Binary size overhead        | ~3 KB                 | ~15-25 KB               |
| Speed (small objects)       | Fast                  | Similar                 |
| Speed (large arrays)        | Faster                | Slower (string alloc)   |
| Debugging ease              | Harder (opaque)       | Easy (readable JSON)    |

## Deriving Serialize and Deserialize

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct GameState {
    score: u64,
    level: u32,
    player: Player,
    inventory: Vec<Item>,
    settings: HashMap<String, String>,
}

#[derive(Serialize, Deserialize, Debug)]
struct Player {
    name: String,
    position: (f64, f64),
    health: f32,
}

#[derive(Serialize, Deserialize, Debug)]
struct Item {
    id: u32,
    name: String,
    quantity: u32,
}
```

This single `#[derive]` generates all the code needed to convert to/from any serde-supported format.

## Enum Serialization Strategies

Serde supports multiple enum representations via attributes:

```rust
// Default: externally tagged
#[derive(Serialize)]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"Circle": {"radius": 5.0}}

// Internally tagged
#[derive(Serialize)]
#[serde(tag = "type")]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"type": "Circle", "radius": 5.0}

// Adjacently tagged
#[derive(Serialize)]
#[serde(tag = "type", content = "data")]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"type": "Circle", "data": {"radius": 5.0}}

// Untagged
#[derive(Serialize)]
#[serde(untagged)]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"radius": 5.0}  (no tag — inferred on deserialize)
```

### Comparison Table

| Strategy         | JSON Output                              | JS-Friendly? | Disambiguates? |
|-----------------|------------------------------------------|---------------|----------------|
| External (default) | `{"Variant": {...}}`                  | Awkward       | Yes            |
| Internal `tag`  | `{"type": "Variant", ...}`               | Natural       | Yes            |
| Adjacent        | `{"type": "Variant", "data": {...}}`     | Clear         | Yes            |
| Untagged        | `{...}` (fields only)                    | Cleanest      | Not always     |

> For JavaScript interop, **internal tagging** (`#[serde(tag = "type")]`) is usually the best choice — it matches TypeScript discriminated unions.

## Handling Option and Result

```rust
#[derive(Serialize, Deserialize)]
struct UserProfile {
    name: String,
    bio: Option<String>,          // null or missing in JSON
    #[serde(default)]
    verified: bool,               // defaults to false if missing
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,   // omitted entirely if None
}
```

```
  Serialization behavior:

  Field                 Value          JSON Output
  ─────────────────────────────────────────────────
  bio                   Some("Hi")     "bio": "Hi"
  bio                   None           "bio": null
  avatar_url            Some("url")    "avatar_url": "url"
  avatar_url            None           (field omitted)
  verified (missing)    -              defaults to false
```

### Common Serde Attributes

| Attribute                               | Effect                                       |
|-----------------------------------------|----------------------------------------------|
| `#[serde(rename = "camelCase")]`        | Rename a single field                        |
| `#[serde(rename_all = "camelCase")]`    | Rename all fields (on struct/enum)           |
| `#[serde(default)]`                     | Use `Default::default()` if field is missing |
| `#[serde(skip)]`                        | Don't serialize or deserialize this field    |
| `#[serde(skip_serializing_if = "...")]` | Conditionally skip during serialization      |
| `#[serde(flatten)]`                     | Inline a nested struct's fields              |
| `#[serde(with = "module")]`             | Custom (de)serialization for a field         |
| `#[serde(deny_unknown_fields)]`         | Error on unexpected JSON keys                |

## Nested and Complex Types

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct Dashboard {
    user_name: String,                     // -> "userName"
    widgets: Vec<Widget>,
    layout: HashMap<String, Position>,
    #[serde(flatten)]
    metadata: Metadata,                    // fields inlined
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "kind")]
enum Widget {
    Chart { data_source: String, chart_type: String },
    Table { columns: Vec<String>, row_count: u32 },
    Text  { content: String },
}

#[derive(Serialize, Deserialize)]
struct Position { x: i32, y: i32, w: u32, h: u32 }

#[derive(Serialize, Deserialize)]
struct Metadata {
    created_at: String,
    version: u32,
}
```

Resulting JSON:
```json
{
  "userName": "alice",
  "widgets": [
    {"kind": "Chart", "data_source": "api/sales", "chart_type": "bar"},
    {"kind": "Text", "content": "Welcome!"}
  ],
  "layout": {
    "widget-0": {"x": 0, "y": 0, "w": 6, "h": 4}
  },
  "created_at": "2025-01-01",
  "version": 3
}
```

## Performance: Minimizing Serialization Cost

### 1. Avoid Unnecessary Copies

```rust
// BAD — serializes entire game state every frame
fn update(state: &GameState) -> JsValue {
    serde_wasm_bindgen::to_value(state).unwrap()
}

// BETTER — only serialize what changed
#[derive(Serialize)]
struct StateDelta {
    updated_entities: Vec<EntityUpdate>,
    removed_ids: Vec<u32>,
}
```

### 2. Use Binary Formats for Large Data

```rust
// JSON: human-readable but large
let json = serde_json::to_vec(&data)?;    // ~500 bytes

// bincode: compact binary format
let bin = bincode::serialize(&data)?;      // ~120 bytes

// Transfer as Uint8Array through data channel or postMessage
```

### 3. Pre-allocate Buffers

```rust
// BAD — allocates a new String each call
fn serialize(data: &Data) -> String {
    serde_json::to_string(data).unwrap()
}

// BETTER — reuse a buffer
fn serialize_into(data: &Data, buf: &mut Vec<u8>) {
    buf.clear();
    serde_json::to_writer(buf, data).unwrap();
}
```

### Format Comparison for Wasm

| Format         | Human Readable | Size     | Speed   | Wasm Binary Overhead |
|---------------|----------------|----------|---------|----------------------|
| serde_json    | Yes            | Large    | Medium  | ~20 KB               |
| serde-wasm-bindgen | N/A (direct) | N/A  | Fast    | ~3 KB                |
| bincode       | No             | Small    | Fast    | ~5 KB                |
| postcard      | No             | Smallest | Fastest | ~3 KB                |
| rmp (msgpack) | No             | Small    | Fast    | ~8 KB                |

## Error Handling

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_config(val: JsValue) -> Result<JsValue, JsValue> {
    // from_value returns a clear error if the shape doesn't match
    let config: Config = serde_wasm_bindgen::from_value(val)
        .map_err(|e| JsValue::from_str(&format!("Invalid config: {}", e)))?;

    // Process...
    let result = process(config);

    serde_wasm_bindgen::to_value(&result)
        .map_err(|e| JsValue::from_str(&format!("Serialization failed: {}", e)))
}
```

## Summary

For Wasm projects, `serde-wasm-bindgen` is the recommended default — it converts directly between Rust types and JavaScript values with minimal overhead. Use `serde_json` when you need human-readable output or are passing data as strings (e.g., `localStorage`, HTTP bodies). Use `#[serde(tag = "type")]` for enums and `#[serde(rename_all = "camelCase")]` for JS-friendly field names. For large data transfers, consider binary formats like `bincode` or `postcard`.
