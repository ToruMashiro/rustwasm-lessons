---
title: "Rust Enums in Wasm"
slug: rust-enums-wasm
difficulty: intermediate
tags: [data-structures]
order: 31
description: Use Rust enums across the Wasm boundary — C-style enums as numbers, complex enums via serde, wasm_bindgen limitations, serde-wasm-bindgen workarounds, and TypeScript type generation.
starter_code: |
  use std::fmt;
  use std::collections::HashMap;

  fn main() {
      println!("=== Rust Enums in Wasm ===\n");

      // --- C-style enums (unit variants) ---
      println!("--- C-Style Enums (map to numbers) ---");

      #[derive(Debug, Clone, Copy, PartialEq)]
      #[repr(u8)]
      enum Color {
          Red = 0,
          Green = 1,
          Blue = 2,
          Yellow = 3,
      }

      impl fmt::Display for Color {
          fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
              match self {
                  Color::Red => write!(f, "Red"),
                  Color::Green => write!(f, "Green"),
                  Color::Blue => write!(f, "Blue"),
                  Color::Yellow => write!(f, "Yellow"),
              }
          }
      }

      impl Color {
          fn from_u8(val: u8) -> Option<Color> {
              match val {
                  0 => Some(Color::Red),
                  1 => Some(Color::Green),
                  2 => Some(Color::Blue),
                  3 => Some(Color::Yellow),
                  _ => None,
              }
          }
      }

      let colors = [Color::Red, Color::Green, Color::Blue, Color::Yellow];
      for c in &colors {
          println!("  {} => numeric value: {}", c, *c as u8);
      }

      println!("\n  Round-trip from number:");
      for i in 0..=4 {
          match Color::from_u8(i) {
              Some(c) => println!("    {} -> {}", i, c),
              None => println!("    {} -> invalid!", i),
          }
      }

      // --- Complex enums (with data) ---
      println!("\n--- Complex Enums (with associated data) ---");

      #[derive(Debug, Clone)]
      enum Shape {
          Circle { radius: f64 },
          Rectangle { width: f64, height: f64 },
          Triangle { base: f64, height: f64 },
          Point,
      }

      impl Shape {
          fn area(&self) -> f64 {
              match self {
                  Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
                  Shape::Rectangle { width, height } => width * height,
                  Shape::Triangle { base, height } => 0.5 * base * height,
                  Shape::Point => 0.0,
              }
          }

          fn name(&self) -> &str {
              match self {
                  Shape::Circle { .. } => "Circle",
                  Shape::Rectangle { .. } => "Rectangle",
                  Shape::Triangle { .. } => "Triangle",
                  Shape::Point => "Point",
              }
          }
      }

      impl fmt::Display for Shape {
          fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
              match self {
                  Shape::Circle { radius } =>
                      write!(f, "Circle(r={})", radius),
                  Shape::Rectangle { width, height } =>
                      write!(f, "Rect({}x{})", width, height),
                  Shape::Triangle { base, height } =>
                      write!(f, "Tri(b={}, h={})", base, height),
                  Shape::Point =>
                      write!(f, "Point"),
              }
          }
      }

      let shapes = vec![
          Shape::Circle { radius: 5.0 },
          Shape::Rectangle { width: 4.0, height: 6.0 },
          Shape::Triangle { base: 3.0, height: 8.0 },
          Shape::Point,
      ];

      for s in &shapes {
          println!("  {} => area = {:.2}", s, s.area());
      }

      // --- Simulating JSON serialization (like serde) ---
      println!("\n--- Serialization Simulation ---");

      impl Shape {
          fn to_json(&self) -> String {
              match self {
                  Shape::Circle { radius } =>
                      format!(r#"{{"type":"Circle","radius":{}}}"#, radius),
                  Shape::Rectangle { width, height } =>
                      format!(r#"{{"type":"Rectangle","width":{},"height":{}}}"#, width, height),
                  Shape::Triangle { base, height } =>
                      format!(r#"{{"type":"Triangle","base":{},"height":{}}}"#, base, height),
                  Shape::Point =>
                      format!(r#""Point""#),
                  }
          }

          fn from_json(json: &str) -> Option<Shape> {
              if json.contains("Circle") {
                  let r: f64 = json.split("radius\":")
                      .nth(1)?.split('}').next()?.parse().ok()?;
                  Some(Shape::Circle { radius: r })
              } else if json.contains("Rectangle") {
                  let parts: Vec<&str> = json.split(':').collect();
                  let w: f64 = parts.get(2)?.split(',').next()?.parse().ok()?;
                  let h: f64 = parts.get(3)?.split('}').next()?.parse().ok()?;
                  Some(Shape::Rectangle { width: w, height: h })
              } else if json == "\"Point\"" {
                  Some(Shape::Point)
              } else {
                  None
              }
          }
      }

      println!("  Serialized:");
      for s in &shapes {
          println!("    {} -> {}", s.name(), s.to_json());
      }

      println!("\n  Round-trip deserialization:");
      let json = r#"{"type":"Circle","radius":3.14}"#;
      if let Some(shape) = Shape::from_json(json) {
          println!("    {} -> {} (area={:.2})", json, shape, shape.area());
      }

      // --- Enum as discriminant + data pattern ---
      println!("\n--- Tagged Union Pattern ---");

      #[derive(Debug)]
      enum Message {
          Text(String),
          Binary(Vec<u8>),
          Close(u16, String),
          Ping,
      }

      impl Message {
          fn discriminant(&self) -> u8 {
              match self {
                  Message::Text(_) => 0,
                  Message::Binary(_) => 1,
                  Message::Close(_, _) => 2,
                  Message::Ping => 3,
              }
          }

          fn describe(&self) -> String {
              match self {
                  Message::Text(t) => format!("Text({} chars)", t.len()),
                  Message::Binary(b) => format!("Binary({} bytes)", b.len()),
                  Message::Close(code, reason) => format!("Close({}, {:?})", code, reason),
                  Message::Ping => "Ping".to_string(),
              }
          }
      }

      let messages = vec![
          Message::Text("Hello, Wasm!".into()),
          Message::Binary(vec![0xDE, 0xAD, 0xBE, 0xEF]),
          Message::Close(1000, "normal".into()),
          Message::Ping,
      ];

      println!("  Messages:");
      for msg in &messages {
          println!("    tag={} | {}", msg.discriminant(), msg.describe());
      }

      // --- Option and Result as enums ---
      println!("\n--- Option & Result (built-in enums) ---");

      fn safe_divide(a: f64, b: f64) -> Result<f64, String> {
          if b == 0.0 {
              Err("division by zero".to_string())
          } else {
              Ok(a / b)
          }
      }

      let test_cases = vec![(10.0, 3.0), (42.0, 0.0), (100.0, 7.0)];
      for (a, b) in test_cases {
          match safe_divide(a, b) {
              Ok(result) => println!("  {}/{} = {:.4}", a, b, result),
              Err(e) => println!("  {}/{} = Error: {}", a, b, e),
          }
      }

      // --- Enum dispatch table ---
      println!("\n--- Enum Dispatch Table ---");

      #[derive(Debug, Clone, Copy, Hash, Eq, PartialEq)]
      enum Operation {
          Add,
          Sub,
          Mul,
          Div,
      }

      let mut dispatch: HashMap<Operation, fn(i32, i32) -> i32> = HashMap::new();
      dispatch.insert(Operation::Add, |a, b| a + b);
      dispatch.insert(Operation::Sub, |a, b| a - b);
      dispatch.insert(Operation::Mul, |a, b| a * b);
      dispatch.insert(Operation::Div, |a, b| if b != 0 { a / b } else { 0 });

      let ops = [Operation::Add, Operation::Sub, Operation::Mul, Operation::Div];
      for op in &ops {
          let func = dispatch[op];
          println!("  {:?}(20, 4) = {}", op, func(20, 4));
      }
  }
expected_output: |
  === Rust Enums in Wasm ===

  --- C-Style Enums (map to numbers) ---
    Red => numeric value: 0
    Green => numeric value: 1
    Blue => numeric value: 2
    Yellow => numeric value: 3

    Round-trip from number:
      0 -> Red
      1 -> Green
      2 -> Blue
      3 -> Yellow
      4 -> invalid!

  --- Complex Enums (with associated data) ---
    Circle(r=5) => area = 78.54
    Rect(4x6) => area = 24.00
    Tri(b=3, h=8) => area = 12.00
    Point => area = 0.00

  --- Serialization Simulation ---
    Serialized:
      Circle -> {"type":"Circle","radius":5}
      Rectangle -> {"type":"Rectangle","width":4,"height":6}
      Triangle -> {"type":"Triangle","base":3,"height":8}
      Point -> "Point"

    Round-trip deserialization:
      {"type":"Circle","radius":3.14} -> Circle(r=3.14) (area=30.97)

  --- Tagged Union Pattern ---
    Messages:
      tag=0 | Text(12 chars)
      tag=1 | Binary(4 bytes)
      tag=2 | Close(1000, "normal")
      tag=3 | Ping

  --- Option & Result (built-in enums) ---
    10/3 = 3.3333
    42/0 = Error: division by zero
    100/7 = 14.2857

  --- Enum Dispatch Table ---
    Add(20, 4) = 24
    Sub(20, 4) = 16
    Mul(20, 4) = 80
    Div(20, 4) = 5
---

## Enums: Rust's Superpower

Rust enums are far more powerful than enums in most languages. They can hold data, have methods, and enforce exhaustive pattern matching at compile time. But crossing the Wasm boundary introduces challenges — JavaScript has no concept of Rust's rich enum types.

## C-Style Enums: The Easy Case

Simple enums without data map directly to JavaScript numbers via `wasm_bindgen`:

```rust
#[wasm_bindgen]
pub enum Direction {
    Up = 0,
    Down = 1,
    Left = 2,
    Right = 3,
}
```

This generates a TypeScript definition:

```typescript
export enum Direction {
    Up = 0,
    Down = 1,
    Left = 2,
    Right = 3,
}
```

**Key rule:** `#[wasm_bindgen]` only supports C-style enums (no data in variants).

### Using repr for Memory Layout

```rust
#[repr(u8)]   // 1 byte
enum Small { A, B, C }

#[repr(u32)]  // 4 bytes, matches JS number
enum JsCompatible { X, Y, Z }
```

```
Memory layout with #[repr(u8)]:
┌────┐
│ 00 │  = A (1 byte)
├────┤
│ 01 │  = B (1 byte)
├────┤
│ 02 │  = C (1 byte)
└────┘

Without repr, Rust chooses the smallest representation.
```

## Complex Enums: The Hard Case

When your enum variants contain data, `wasm_bindgen` can't export them directly:

```rust
// This WON'T compile with #[wasm_bindgen]
enum Shape {
    Circle { radius: f64 },        // has data
    Rectangle { w: f64, h: f64 },  // has data
    Point,                          // no data
}
```

**Error:** `wasm_bindgen` doesn't support enums with data fields. You need a workaround.

## Workaround 1: serde-wasm-bindgen

The most popular solution is to serialize complex enums to `JsValue` using `serde`:

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]  // internally tagged
pub enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Point,
}

#[wasm_bindgen]
pub fn create_shape(kind: &str, params: JsValue) -> JsValue {
    let shape: Shape = match kind {
        "circle" => Shape::Circle {
            radius: serde_wasm_bindgen::from_value(params).unwrap()
        },
        _ => Shape::Point,
    };
    serde_wasm_bindgen::to_value(&shape).unwrap()
}
```

### Serde Tagging Strategies

Serde offers several ways to represent enums in JSON:

| Strategy | Attribute | JSON Output |
|----------|----------|-------------|
| Externally tagged (default) | none | `{"Circle": {"radius": 5}}` |
| Internally tagged | `#[serde(tag = "type")]` | `{"type": "Circle", "radius": 5}` |
| Adjacently tagged | `#[serde(tag = "t", content = "c")]` | `{"t": "Circle", "c": {"radius": 5}}` |
| Untagged | `#[serde(untagged)]` | `{"radius": 5}` |

```
Externally tagged (default):
{ "Circle": { "radius": 5.0 } }
  ~~~~~~~~   ~~~~~~~~~~~~~~~~
  variant     data

Internally tagged (#[serde(tag = "type")]):
{ "type": "Circle", "radius": 5.0 }
  ~~~~~~~~~~~~~~~~  ~~~~~~~~~~~~~~
  discriminant       data (flattened)

Adjacently tagged (#[serde(tag = "t", content = "c")]):
{ "t": "Circle", "c": { "radius": 5.0 } }
  ~~~~~~~~~~~~~~  ~~~~~~~~~~~~~~~~~~~~~~~~
  discriminant     data (nested)
```

## Workaround 2: Manual Conversion with Structs

For performance-sensitive code, avoid serde overhead by manually converting:

```rust
#[wasm_bindgen]
pub struct ShapeData {
    kind: u8,          // 0=Circle, 1=Rect, 2=Point
    param1: f64,       // radius or width
    param2: f64,       // 0 or height
}

#[wasm_bindgen]
impl ShapeData {
    pub fn circle(radius: f64) -> ShapeData {
        ShapeData { kind: 0, param1: radius, param2: 0.0 }
    }

    pub fn rectangle(width: f64, height: f64) -> ShapeData {
        ShapeData { kind: 1, param1: width, param2: height }
    }

    pub fn area(&self) -> f64 {
        match self.kind {
            0 => std::f64::consts::PI * self.param1 * self.param1,
            1 => self.param1 * self.param2,
            _ => 0.0,
        }
    }
}
```

## Workaround 3: TypeScript Union Types via tsify

The `tsify` crate generates proper TypeScript discriminated unions:

```rust
use tsify::Tsify;
use serde::{Serialize, Deserialize};

#[derive(Tsify, Serialize, Deserialize)]
#[tsify(into_wasm_abi, from_wasm_abi)]
#[serde(tag = "type")]
pub enum GameEvent {
    Move { x: f64, y: f64 },
    Attack { target_id: u32, damage: f64 },
    Heal { amount: f64 },
    Disconnect,
}
```

This generates:

```typescript
export type GameEvent =
    | { type: "Move"; x: number; y: number }
    | { type: "Attack"; target_id: number; damage: number }
    | { type: "Heal"; amount: number }
    | { type: "Disconnect" };
```

Now TypeScript's type narrowing works perfectly:

```typescript
function handleEvent(event: GameEvent) {
    switch (event.type) {
        case "Move":
            console.log(`Moving to (${event.x}, ${event.y})`);
            break;
        case "Attack":
            console.log(`Attacking ${event.target_id} for ${event.damage}`);
            break;
    }
}
```

## Performance Comparison

| Method | Serialization Cost | Type Safety | JS Ergonomics |
|--------|-------------------|-------------|---------------|
| C-style enum | None (u32) | Excellent | Good |
| serde-wasm-bindgen | Medium (JSON-like) | Good | Excellent |
| Manual struct | None | Manual | Fair |
| tsify | Medium | Excellent | Excellent |

## Option and Result Across the Boundary

`Option<T>` maps to `T | undefined` in JavaScript for primitive types:

```rust
#[wasm_bindgen]
pub fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".into()) } else { None }
}
```

```typescript
const name: string | undefined = find_user(1); // "Alice"
const none: string | undefined = find_user(99); // undefined
```

`Result<T, E>` throws a JavaScript exception on `Err`:

```rust
#[wasm_bindgen]
pub fn parse_number(s: &str) -> Result<f64, JsError> {
    s.parse::<f64>().map_err(|e| JsError::new(&e.to_string()))
}
```

```typescript
try {
    const n = parse_number("42.5"); // 42.5
    const bad = parse_number("abc"); // throws!
} catch (e) {
    console.error(e.message); // "invalid float literal"
}
```

## Key Takeaways

1. **C-style enums** work directly with `#[wasm_bindgen]` — they map to JS numbers
2. **Complex enums** (with data) need a workaround: `serde-wasm-bindgen`, manual structs, or `tsify`
3. **serde's tagging strategies** control how enum variants appear in JSON — internally tagged is usually best for JS
4. **tsify** generates TypeScript discriminated unions, giving you full type safety on both sides
5. **`Option<T>`** maps to `T | undefined`, **`Result<T, E>`** maps to return-or-throw
6. For hot paths, avoid serde overhead — use C-style enums or manual conversion structs
