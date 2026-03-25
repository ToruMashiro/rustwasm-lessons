---
title: "WasmにおけるRustの列挙型"
slug: rust-enums-wasm
difficulty: intermediate
tags: [data-structures]
order: 31
description: Wasm境界を越えたRust列挙型の使い方 — C言語スタイルの列挙型を数値として扱う方法、serdeによる複雑な列挙型、wasm_bindgenの制限、serde-wasm-bindgenの回避策、TypeScript型生成。
starter_code: |
  use std::fmt;
  use std::collections::HashMap;

  fn main() {
      println!("=== WasmにおけるRust列挙型 ===\n");

      // --- C言語スタイルの列挙型（ユニットバリアント） ---
      println!("--- C言語スタイル列挙型（数値にマッピング） ---");

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
          println!("  {} => 数値: {}", c, *c as u8);
      }

      println!("\n  数値からの往復変換:");
      for i in 0..=4 {
          match Color::from_u8(i) {
              Some(c) => println!("    {} -> {}", i, c),
              None => println!("    {} -> 無効!", i),
          }
      }

      // --- 複雑な列挙型（データ付き） ---
      println!("\n--- 複雑な列挙型（関連データ付き） ---");

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
          println!("  {} => 面積 = {:.2}", s, s.area());
      }

      // --- JSONシリアライゼーションのシミュレーション（serdeのような） ---
      println!("\n--- シリアライゼーションシミュレーション ---");

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

      println!("  シリアライズ結果:");
      for s in &shapes {
          println!("    {} -> {}", s.name(), s.to_json());
      }

      println!("\n  往復デシリアライゼーション:");
      let json = r#"{"type":"Circle","radius":3.14}"#;
      if let Some(shape) = Shape::from_json(json) {
          println!("    {} -> {} (面積={:.2})", json, shape, shape.area());
      }

      // --- 判別子+データパターンとしての列挙型 ---
      println!("\n--- タグ付きユニオンパターン ---");

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
                  Message::Text(t) => format!("Text({}文字)", t.len()),
                  Message::Binary(b) => format!("Binary({}バイト)", b.len()),
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

      println!("  メッセージ:");
      for msg in &messages {
          println!("    tag={} | {}", msg.discriminant(), msg.describe());
      }

      // --- 列挙型としてのOptionとResult ---
      println!("\n--- Option & Result（組み込み列挙型） ---");

      fn safe_divide(a: f64, b: f64) -> Result<f64, String> {
          if b == 0.0 {
              Err("ゼロ除算".to_string())
          } else {
              Ok(a / b)
          }
      }

      let test_cases = vec![(10.0, 3.0), (42.0, 0.0), (100.0, 7.0)];
      for (a, b) in test_cases {
          match safe_divide(a, b) {
              Ok(result) => println!("  {}/{} = {:.4}", a, b, result),
              Err(e) => println!("  {}/{} = エラー: {}", a, b, e),
          }
      }

      // --- 列挙型ディスパッチテーブル ---
      println!("\n--- 列挙型ディスパッチテーブル ---");

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
  === WasmにおけるRust列挙型 ===

  --- C言語スタイル列挙型（数値にマッピング） ---
    Red => 数値: 0
    Green => 数値: 1
    Blue => 数値: 2
    Yellow => 数値: 3

    数値からの往復変換:
      0 -> Red
      1 -> Green
      2 -> Blue
      3 -> Yellow
      4 -> 無効!

  --- 複雑な列挙型（関連データ付き） ---
    Circle(r=5) => 面積 = 78.54
    Rect(4x6) => 面積 = 24.00
    Tri(b=3, h=8) => 面積 = 12.00
    Point => 面積 = 0.00

  --- シリアライゼーションシミュレーション ---
    シリアライズ結果:
      Circle -> {"type":"Circle","radius":5}
      Rectangle -> {"type":"Rectangle","width":4,"height":6}
      Triangle -> {"type":"Triangle","base":3,"height":8}
      Point -> "Point"

    往復デシリアライゼーション:
      {"type":"Circle","radius":3.14} -> Circle(r=3.14) (面積=30.97)

  --- タグ付きユニオンパターン ---
    メッセージ:
      tag=0 | Text(12文字)
      tag=1 | Binary(4バイト)
      tag=2 | Close(1000, "normal")
      tag=3 | Ping

  --- Option & Result（組み込み列挙型） ---
    10/3 = 3.3333
    42/0 = エラー: ゼロ除算
    100/7 = 14.2857

  --- 列挙型ディスパッチテーブル ---
    Add(20, 4) = 24
    Sub(20, 4) = 16
    Mul(20, 4) = 80
    Div(20, 4) = 5
---

## 列挙型：Rustの超能力

Rustの列挙型は多くの言語の列挙型よりはるかに強力です。データを保持でき、メソッドを持ち、コンパイル時に網羅的なパターンマッチングを強制します。しかし、Wasm境界を越えると課題が生じます — JavaScriptにはRustのリッチな列挙型に相当する概念がありません。

## C言語スタイルの列挙型：簡単なケース

データを持たないシンプルな列挙型は、`wasm_bindgen`を通じてJavaScriptの数値に直接マッピングされます：

```rust
#[wasm_bindgen]
pub enum Direction {
    Up = 0,
    Down = 1,
    Left = 2,
    Right = 3,
}
```

これにより以下のTypeScript定義が生成されます：

```typescript
export enum Direction {
    Up = 0,
    Down = 1,
    Left = 2,
    Right = 3,
}
```

**重要なルール：** `#[wasm_bindgen]`はC言語スタイルの列挙型（バリアントにデータなし）のみサポートします。

### reprによるメモリレイアウト

```rust
#[repr(u8)]   // 1バイト
enum Small { A, B, C }

#[repr(u32)]  // 4バイト、JSの数値と一致
enum JsCompatible { X, Y, Z }
```

```
#[repr(u8)]のメモリレイアウト:
┌────┐
│ 00 │  = A（1バイト）
├────┤
│ 01 │  = B（1バイト）
├────┤
│ 02 │  = C（1バイト）
└────┘

reprなしの場合、Rustは最小の表現を選択します。
```

## 複雑な列挙型：難しいケース

列挙型のバリアントにデータが含まれる場合、`wasm_bindgen`は直接エクスポートできません：

```rust
// これは#[wasm_bindgen]ではコンパイルできない
enum Shape {
    Circle { radius: f64 },        // データあり
    Rectangle { w: f64, h: f64 },  // データあり
    Point,                          // データなし
}
```

**エラー：** `wasm_bindgen`はデータフィールドを持つ列挙型をサポートしていません。回避策が必要です。

## 回避策1：serde-wasm-bindgen

最も人気のある解決策は、`serde`を使って複雑な列挙型を`JsValue`にシリアライズすることです：

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]  // 内部タグ付き
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

### Serdeのタグ付け戦略

SerdeはJSONでの列挙型表現にいくつかの方法を提供します：

| 戦略 | 属性 | JSON出力 |
|----------|----------|-------------|
| 外部タグ付き（デフォルト） | なし | `{"Circle": {"radius": 5}}` |
| 内部タグ付き | `#[serde(tag = "type")]` | `{"type": "Circle", "radius": 5}` |
| 隣接タグ付き | `#[serde(tag = "t", content = "c")]` | `{"t": "Circle", "c": {"radius": 5}}` |
| タグなし | `#[serde(untagged)]` | `{"radius": 5}` |

```
外部タグ付き（デフォルト）:
{ "Circle": { "radius": 5.0 } }
  ~~~~~~~~   ~~~~~~~~~~~~~~~~
  バリアント   データ

内部タグ付き（#[serde(tag = "type")]）:
{ "type": "Circle", "radius": 5.0 }
  ~~~~~~~~~~~~~~~~  ~~~~~~~~~~~~~~
  判別子             データ（フラット化）

隣接タグ付き（#[serde(tag = "t", content = "c")]）:
{ "t": "Circle", "c": { "radius": 5.0 } }
  ~~~~~~~~~~~~~~  ~~~~~~~~~~~~~~~~~~~~~~~~
  判別子           データ（ネスト）
```

## 回避策2：構造体による手動変換

パフォーマンスが重要なコードでは、serdeのオーバーヘッドを避けて手動変換します：

```rust
#[wasm_bindgen]
pub struct ShapeData {
    kind: u8,          // 0=Circle, 1=Rect, 2=Point
    param1: f64,       // radiusまたはwidth
    param2: f64,       // 0またはheight
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

## 回避策3：tsifyによるTypeScriptユニオン型

`tsify`クレートは適切なTypeScript判別共用体を生成します：

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

これにより以下が生成されます：

```typescript
export type GameEvent =
    | { type: "Move"; x: number; y: number }
    | { type: "Attack"; target_id: number; damage: number }
    | { type: "Heal"; amount: number }
    | { type: "Disconnect" };
```

TypeScriptの型ナローイングが完璧に機能します：

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

## パフォーマンス比較

| メソッド | シリアライゼーションコスト | 型安全性 | JS側の使いやすさ |
|--------|-------------------|-------------|---------------|
| C言語スタイル列挙型 | なし（u32） | 優秀 | 良好 |
| serde-wasm-bindgen | 中程度（JSON風） | 良好 | 優秀 |
| 手動構造体 | なし | 手動 | 普通 |
| tsify | 中程度 | 優秀 | 優秀 |

## 境界を越えるOptionとResult

`Option<T>`はプリミティブ型の場合、JavaScriptで`T | undefined`にマッピングされます：

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

`Result<T, E>`は`Err`の場合にJavaScript例外をスローします：

```rust
#[wasm_bindgen]
pub fn parse_number(s: &str) -> Result<f64, JsError> {
    s.parse::<f64>().map_err(|e| JsError::new(&e.to_string()))
}
```

```typescript
try {
    const n = parse_number("42.5"); // 42.5
    const bad = parse_number("abc"); // スロー！
} catch (e) {
    console.error(e.message); // "invalid float literal"
}
```

## まとめ

1. **C言語スタイルの列挙型**は`#[wasm_bindgen]`で直接使える — JSの数値にマッピングされる
2. **複雑な列挙型**（データ付き）には回避策が必要：`serde-wasm-bindgen`、手動構造体、または`tsify`
3. **serdeのタグ付け戦略**はJSONでの列挙型バリアントの表現方法を制御する — JSには内部タグ付きが通常最適
4. **tsify**はTypeScript判別共用体を生成し、両側で完全な型安全性を提供する
5. **`Option<T>`**は`T | undefined`に、**`Result<T, E>`**は戻り値またはスローにマッピングされる
6. ホットパスではserdeのオーバーヘッドを避ける — C言語スタイル列挙型か手動変換構造体を使用する
