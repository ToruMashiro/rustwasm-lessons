---
title: Wasm向けSerdeシリアライゼーション
slug: serde-wasm
difficulty: intermediate
tags: [data-structures]
order: 37
description: Wasmにおけるserdeシリアライゼーションをマスターします — serde-wasm-bindgenとJsValueの比較、複雑な型のシリアライズ、パフォーマンスの最適化。
starter_code: |
  // 純粋なRustでserdeシリアライゼーションのコンセプトをデモ
  // serde_jsonを使用（同じパターンがserde-wasm-bindgenにも適用可能）

  use std::collections::HashMap;
  use std::fmt;

  // --- serdeクレートなしでのシミュレーション ---
  // 実際のコードでは #[derive(Serialize, Deserialize)] を使用

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

  // 手動JSONシリアライゼーション（serdeが自動で行うもの）
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
      println!("=== Serdeシリアライゼーション デモ ===\n");

      // テストデータの作成
      let alice = User::from_fields("Alice", 30, Some("alice@example.com"), Role::Admin);
      let bob = User::from_fields("Bob", 25, None, Role::Viewer);

      // 個別ユーザーのシリアライズ
      println!("Alice JSON:");
      println!("  {}", alice.to_json());
      println!();
      println!("Bob JSON:");
      println!("  {}", bob.to_json());
      println!();

      // 複雑なネストされたレスポンスのシリアライズ
      let mut metadata = HashMap::new();
      metadata.insert("page".to_string(), "1".to_string());
      metadata.insert("total".to_string(), "2".to_string());

      let response = ApiResponse {
          status: 200,
          data: vec![alice.clone(), bob.clone()],
          metadata,
      };

      println!("APIレスポンス JSON:");
      println!("  {}", response.to_json());
      println!();

      // Optionの処理をデモ
      println!("Option<String>のシリアライゼーション:");
      println!("  Some(email) -> {}", format!(r#""{}""#, alice.email.as_ref().unwrap()));
      println!("  None        -> null");
      println!();

      // enumのシリアライゼーションをデモ
      println!("enumバリアント:");
      println!("  Admin  -> {}", Role::Admin);
      println!("  Editor -> {}", Role::Editor);
      println!("  Viewer -> {}", Role::Viewer);
  }
expected_output: |
  === Serdeシリアライゼーション デモ ===

  Alice JSON:
    {"name":"Alice","age":30,"email":"alice@example.com","role":"admin"}

  Bob JSON:
    {"name":"Bob","age":25,"email":null,"role":"viewer"}

  APIレスポンス JSON:
    {"status":200,"data":[{"name":"Alice","age":30,"email":"alice@example.com","role":"admin"},{"name":"Bob","age":25,"email":null,"role":"viewer"}],"metadata":{"page":"1","total":"2"}}

  Option<String>のシリアライゼーション:
    Some(email) -> "alice@example.com"
    None        -> null

  enumバリアント:
    Admin  -> "admin"
    Editor -> "editor"
    Viewer -> "viewer"
---

## WasmでSerdeが重要な理由

Rust/WasmがJavaScriptと通信する際、データは**Wasm境界**を越える必要があります。Rustの構造体はリニアメモリに存在し、JavaScriptオブジェクトはJSヒープに存在します。Serdeは2つの表現間を変換することで、このギャップを橋渡しします。

```
  Rust/Wasm側                 JS側
  ┌────────────────┐          ┌────────────────┐
  │ struct User {  │          │ { name: "Ali", │
  │   name: String │  serde   │   age: 30,     │
  │   age: u32     │ <======> │   role: "admin"}│
  │   role: Role   │          │                │
  │ }              │          │ (JSオブジェクト) │
  └────────────────┘          └────────────────┘
       リニアメモリ                  JSヒープ
```

## 2つのアプローチ: serde-wasm-bindgen と serde_json

### アプローチ1: serde-wasm-bindgen（推奨）

JSON文字列を経由せず、Rustの型と`JsValue`を直接変換します：

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

### アプローチ2: serde_json（JsValue文字列経由）

JSON文字列にシリアライズし、反対側でパースします：

```rust
#[wasm_bindgen]
pub fn process_point_json(json: &str) -> String {
    let point: Point = serde_json::from_str(json).unwrap();
    let result = Point { x: point.x * 2.0, y: point.y * 2.0 };
    serde_json::to_string(&result).unwrap()
}
```

### 比較

```
  serde-wasm-bindgenの経路:
  Rust構造体 ──> JsValue  （直接、1ステップ）

  serde_jsonの経路:
  Rust構造体 ──> JSON文字列 ──> JS JSON.parse() ──> JSオブジェクト
                     ^                ^
                シリアライズ        パース
                 (Rust)           (JSエンジン)
```

| 特徴                        | serde-wasm-bindgen    | serde_json              |
|-----------------------------|-----------------------|-------------------------|
| 中間形式                    | なし（直接）          | JSON文字列              |
| Map, Set, Dateのサポート    | あり（ネイティブJS型） | なし（JSONサブセットのみ）|
| BigIntサポート              | あり                  | なし                    |
| undefined vs nullの区別     | あり                  | なし（両方nullになる）   |
| バイナリサイズのオーバーヘッド | ~3 KB               | ~15-25 KB               |
| 速度（小さなオブジェクト）   | 高速                 | 同程度                  |
| 速度（大きな配列）           | より高速             | 低速（文字列アロケーション）|
| デバッグの容易さ            | 難しい（不透明）      | 容易（可読なJSON）       |

## SerializeとDeserializeのderive

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

この単一の`#[derive]`で、serde対応のあらゆる形式への変換に必要なコードがすべて生成されます。

## enumのシリアライゼーション戦略

Serdeは属性を使った複数のenum表現をサポートしています：

```rust
// デフォルト: 外部タグ付き
#[derive(Serialize)]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"Circle": {"radius": 5.0}}

// 内部タグ付き
#[derive(Serialize)]
#[serde(tag = "type")]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"type": "Circle", "radius": 5.0}

// 隣接タグ付き
#[derive(Serialize)]
#[serde(tag = "type", content = "data")]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"type": "Circle", "data": {"radius": 5.0}}

// タグなし
#[derive(Serialize)]
#[serde(untagged)]
enum Shape {
    Circle { radius: f64 },
    Rect { w: f64, h: f64 },
}
// -> {"radius": 5.0}  （タグなし — デシリアライズ時に推論）
```

### 比較表

| 戦略             | JSON出力                                 | JS親和性？    | 識別可能？     |
|-----------------|------------------------------------------|---------------|----------------|
| 外部（デフォルト） | `{"Variant": {...}}`                   | 扱いにくい    | はい           |
| 内部 `tag`      | `{"type": "Variant", ...}`               | 自然          | はい           |
| 隣接            | `{"type": "Variant", "data": {...}}`     | 明確          | はい           |
| タグなし         | `{...}`（フィールドのみ）                | 最もクリーン  | 常にではない   |

> JavaScriptとの相互運用では、**内部タグ付き**（`#[serde(tag = "type")]`）が通常最良の選択です — TypeScriptの判別共用体に一致します。

## OptionとResultの処理

```rust
#[derive(Serialize, Deserialize)]
struct UserProfile {
    name: String,
    bio: Option<String>,          // JSONでnullまたは欠落
    #[serde(default)]
    verified: bool,               // 欠落時はfalseにデフォルト
    #[serde(skip_serializing_if = "Option::is_none")]
    avatar_url: Option<String>,   // Noneの場合は完全に省略
}
```

```
  シリアライゼーション動作:

  フィールド            値             JSON出力
  ─────────────────────────────────────────────────
  bio                   Some("Hi")     "bio": "Hi"
  bio                   None           "bio": null
  avatar_url            Some("url")    "avatar_url": "url"
  avatar_url            None           （フィールド省略）
  verified（欠落時）     -              falseにデフォルト
```

### よく使うSerde属性

| 属性                                    | 効果                                         |
|-----------------------------------------|----------------------------------------------|
| `#[serde(rename = "camelCase")]`        | 単一フィールドのリネーム                      |
| `#[serde(rename_all = "camelCase")]`    | すべてのフィールドをリネーム（構造体/enum）   |
| `#[serde(default)]`                     | フィールド欠落時に`Default::default()`を使用  |
| `#[serde(skip)]`                        | シリアライズもデシリアライズもしない          |
| `#[serde(skip_serializing_if = "...")]` | シリアライゼーション時に条件付きでスキップ    |
| `#[serde(flatten)]`                     | ネストされた構造体のフィールドをインライン化   |
| `#[serde(with = "module")]`             | フィールドのカスタム（デ）シリアライゼーション |
| `#[serde(deny_unknown_fields)]`         | 予期しないJSONキーでエラー                    |

## ネストされた複雑な型

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct Dashboard {
    user_name: String,                     // -> "userName"
    widgets: Vec<Widget>,
    layout: HashMap<String, Position>,
    #[serde(flatten)]
    metadata: Metadata,                    // フィールドがインライン化
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

結果のJSON：
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

## パフォーマンス: シリアライゼーションコストの最小化

### 1. 不要なコピーを避ける

```rust
// 悪い例 — 毎フレームゲーム状態全体をシリアライズ
fn update(state: &GameState) -> JsValue {
    serde_wasm_bindgen::to_value(state).unwrap()
}

// 良い例 — 変更されたものだけをシリアライズ
#[derive(Serialize)]
struct StateDelta {
    updated_entities: Vec<EntityUpdate>,
    removed_ids: Vec<u32>,
}
```

### 2. 大きなデータにはバイナリ形式を使う

```rust
// JSON: 人間が読めるが大きい
let json = serde_json::to_vec(&data)?;    // ~500バイト

// bincode: コンパクトなバイナリ形式
let bin = bincode::serialize(&data)?;      // ~120バイト

// Uint8ArrayとしてデータチャネルやpostMessageで転送
```

### 3. バッファの事前アロケート

```rust
// 悪い例 — 呼び出しごとに新しいStringをアロケート
fn serialize(data: &Data) -> String {
    serde_json::to_string(data).unwrap()
}

// 良い例 — バッファを再利用
fn serialize_into(data: &Data, buf: &mut Vec<u8>) {
    buf.clear();
    serde_json::to_writer(buf, data).unwrap();
}
```

### Wasm向けフォーマット比較

| フォーマット     | 人間が読める | サイズ   | 速度    | Wasmバイナリオーバーヘッド |
|---------------|----------------|----------|---------|----------------------|
| serde_json    | はい           | 大きい   | 中程度  | ~20 KB               |
| serde-wasm-bindgen | N/A（直接） | N/A  | 高速    | ~3 KB                |
| bincode       | いいえ         | 小さい   | 高速    | ~5 KB                |
| postcard      | いいえ         | 最小     | 最速    | ~3 KB                |
| rmp (msgpack) | いいえ         | 小さい   | 高速    | ~8 KB                |

## エラーハンドリング

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_config(val: JsValue) -> Result<JsValue, JsValue> {
    // from_valueは形状が一致しない場合に明確なエラーを返す
    let config: Config = serde_wasm_bindgen::from_value(val)
        .map_err(|e| JsValue::from_str(&format!("無効な設定: {}", e)))?;

    // 処理...
    let result = process(config);

    serde_wasm_bindgen::to_value(&result)
        .map_err(|e| JsValue::from_str(&format!("シリアライゼーション失敗: {}", e)))
}
```

## まとめ

Wasmプロジェクトでは、`serde-wasm-bindgen`が推奨デフォルトです — 最小限のオーバーヘッドでRustの型とJavaScriptの値を直接変換します。人間が読める出力が必要な場合や、データを文字列として渡す場合（例：`localStorage`、HTTPボディ）は`serde_json`を使用しましょう。enumには`#[serde(tag = "type")]`を、JS親和的なフィールド名には`#[serde(rename_all = "camelCase")]`を使用しましょう。大量のデータ転送には`bincode`や`postcard`などのバイナリ形式を検討してください。
