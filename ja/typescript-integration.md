---
title: "Wasm + TypeScript 統合"
slug: typescript-integration
difficulty: intermediate
tags: [getting-started]
order: 39
description: RustとTypeScriptを強力な型で橋渡し — wasm-bindgenが.d.tsファイルを生成する仕組み、tsifyを使った複雑な型の扱い、TypeScriptユーザーにネイティブに感じられる型安全なWasm APIの構築方法を学びます。
starter_code: |
  use std::fmt;

  // プレーンRustで型安全なインターフェースパターンをデモします。
  // 実際のWasmプロジェクトでは、wasm-bindgen + tsifyが
  // TypeScript定義を自動生成します。ここでは同じ概念をモデル化します。

  // --- TypeScript風の型定義をシミュレーション ---

  #[derive(Debug, Clone)]
  enum TypeScriptType {
      Number,
      String,
      Boolean,
      Array(Box<TypeScriptType>),
      Option(Box<TypeScriptType>),
      Object(Vec<(String, TypeScriptType)>),
      Union(Vec<TypeScriptType>),
      Void,
  }

  impl fmt::Display for TypeScriptType {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          match self {
              TypeScriptType::Number => write!(f, "number"),
              TypeScriptType::String => write!(f, "string"),
              TypeScriptType::Boolean => write!(f, "boolean"),
              TypeScriptType::Array(inner) => write!(f, "{}[]", inner),
              TypeScriptType::Option(inner) => write!(f, "{} | undefined", inner),
              TypeScriptType::Object(fields) => {
                  write!(f, "{{ ")?;
                  for (i, (name, ty)) in fields.iter().enumerate() {
                      if i > 0 { write!(f, "; ")?; }
                      write!(f, "{}: {}", name, ty)?;
                  }
                  write!(f, " }}")
              }
              TypeScriptType::Union(types) => {
                  for (i, ty) in types.iter().enumerate() {
                      if i > 0 { write!(f, " | ")?; }
                      write!(f, "{}", ty)?;
                  }
                  Ok(())
              }
              TypeScriptType::Void => write!(f, "void"),
          }
      }
  }

  // --- 関数シグネチャ生成器（wasm-bindgenと同様の動作） ---

  struct FunctionBinding {
      name: String,
      params: Vec<(String, TypeScriptType)>,
      return_type: TypeScriptType,
  }

  impl FunctionBinding {
      fn to_ts_declaration(&self) -> String {
          let params: Vec<String> = self.params.iter()
              .map(|(name, ty)| format!("{}: {}", name, ty))
              .collect();
          format!("export function {}({}): {};",
              self.name, params.join(", "), self.return_type)
      }
  }

  // --- インターフェース生成器（tsifyが構造体/列挙型に対して行う処理と同様） ---

  struct InterfaceBinding {
      name: String,
      fields: Vec<(String, TypeScriptType, bool)>, // (名前, 型, オプショナル)
  }

  impl InterfaceBinding {
      fn to_ts_interface(&self) -> String {
          let mut lines = vec![format!("export interface {} {{", self.name)];
          for (name, ty, optional) in &self.fields {
              let opt = if *optional { "?" } else { "" };
              lines.push(format!("  {}{}: {};", name, opt, ty));
          }
          lines.push("}".to_string());
          lines.join("\n")
      }
  }

  // --- 列挙型からTypeScriptユニオン型への変換 ---

  struct EnumBinding {
      name: String,
      variants: Vec<(String, Option<Vec<(String, TypeScriptType)>>)>,
  }

  impl EnumBinding {
      fn to_ts_type(&self) -> String {
          let variants: Vec<String> = self.variants.iter().map(|(name, fields)| {
              match fields {
                  None => format!("{{ tag: \"{}\" }}", name),
                  Some(f) => {
                      let field_strs: Vec<String> = f.iter()
                          .map(|(n, t)| format!("{}: {}", n, t))
                          .collect();
                      format!("{{ tag: \"{}\"; {} }}", name, field_strs.join("; "))
                  }
              }
          }).collect();
          format!("export type {} = {};", self.name, variants.join("\n  | "))
      }
  }

  // --- 完全な .d.ts ファイル生成のデモ ---

  fn generate_dts_file() -> String {
      let mut declarations = Vec::new();

      declarations.push("// wasm-bindgen + tsify により自動生成".to_string());
      declarations.push("// 手動で編集しないでください!\n".to_string());

      // User構造体のインターフェース
      let user = InterfaceBinding {
          name: "User".to_string(),
          fields: vec![
              ("id".into(), TypeScriptType::Number, false),
              ("name".into(), TypeScriptType::String, false),
              ("email".into(), TypeScriptType::String, true),
              ("roles".into(), TypeScriptType::Array(Box::new(TypeScriptType::String)), false),
          ],
      };
      declarations.push(user.to_ts_interface());

      // Configのインターフェース
      let config = InterfaceBinding {
          name: "AppConfig".to_string(),
          fields: vec![
              ("debug".into(), TypeScriptType::Boolean, false),
              ("max_items".into(), TypeScriptType::Number, false),
              ("title".into(), TypeScriptType::String, true),
          ],
      };
      declarations.push(config.to_ts_interface());

      // 列挙型 -> タグ付きユニオン
      let status = EnumBinding {
          name: "Status".to_string(),
          variants: vec![
              ("Loading".to_string(), None),
              ("Ready".to_string(), Some(vec![
                  ("data".into(), TypeScriptType::Array(Box::new(TypeScriptType::Number))),
              ])),
              ("Error".to_string(), Some(vec![
                  ("message".into(), TypeScriptType::String),
              ])),
          ],
      };
      declarations.push(status.to_ts_type());

      // 関数バインディング
      let functions = vec![
          FunctionBinding {
              name: "greet".to_string(),
              params: vec![("name".into(), TypeScriptType::String)],
              return_type: TypeScriptType::String,
          },
          FunctionBinding {
              name: "process_users".to_string(),
              params: vec![
                  ("users".into(), TypeScriptType::Array(Box::new(
                      TypeScriptType::Object(vec![
                          ("id".into(), TypeScriptType::Number),
                          ("name".into(), TypeScriptType::String),
                      ])
                  ))),
                  ("config".into(), TypeScriptType::Object(vec![
                      ("debug".into(), TypeScriptType::Boolean),
                  ])),
              ],
              return_type: TypeScriptType::Number,
          },
          FunctionBinding {
              name: "get_status".to_string(),
              params: vec![],
              return_type: TypeScriptType::Union(vec![
                  TypeScriptType::String,
                  TypeScriptType::Number,
              ]),
          },
      ];

      for func in &functions {
          declarations.push(func.to_ts_declaration());
      }

      declarations.join("\n\n")
  }

  // --- 型マッピングテーブル ---

  fn print_type_mapping() {
      println!("=== Rust -> TypeScript 型マッピング ===\n");
      let mappings = vec![
          ("i8, i16, i32, u8, u16, u32, f32, f64", "number"),
          ("i64, u64", "bigint"),
          ("bool", "boolean"),
          ("String, &str", "string"),
          ("Option<T>", "T | undefined"),
          ("Vec<T>", "T[]"),
          ("HashMap<K, V>", "Map<K, V>"),
          ("Result<T, E>", "T  (Errの場合throw)"),
          ("()", "void"),
          ("#[wasm_bindgen] struct", "class (メソッド付き)"),
          ("#[tsify] struct", "interface (プレーンオブジェクト)"),
          ("#[tsify] enum", "タグ付きユニオン"),
      ];

      println!("{:<40} {}", "Rust型", "TypeScript型");
      println!("{}", "-".repeat(65));
      for (rust_ty, ts_ty) in &mappings {
          println!("{:<40} {}", rust_ty, ts_ty);
      }
  }

  fn main() {
      print_type_mapping();

      println!("\n\n=== 生成された .d.ts ファイル ===\n");
      let dts = generate_dts_file();
      println!("{}", dts);
  }
expected_output: |
  === Rust -> TypeScript 型マッピング ===

  Rust型                                   TypeScript型
  -----------------------------------------------------------------
  i8, i16, i32, u8, u16, u32, f32, f64     number
  i64, u64                               bigint
  bool                                   boolean
  String, &str                           string
  Option<T>                              T | undefined
  Vec<T>                                 T[]
  HashMap<K, V>                          Map<K, V>
  Result<T, E>                           T  (Errの場合throw)
  ()                                     void
  #[wasm_bindgen] struct                 class (メソッド付き)
  #[tsify] struct                        interface (プレーンオブジェクト)
  #[tsify] enum                          タグ付きユニオン


  === 生成された .d.ts ファイル ===

  // wasm-bindgen + tsify により自動生成

  // 手動で編集しないでください!


  export interface User {
    id: number;
    name: string;
    email?: string;
    roles: string[];
  }

  export interface AppConfig {
    debug: boolean;
    max_items: number;
    title?: string;
  }

  export type Status = { tag: "Loading" }
    | { tag: "Ready"; data: number[] }
    | { tag: "Error"; message: string };

  export function greet(name: string): string;

  export function process_users(users: { id: number; name: string }[], config: { debug: boolean }): number;

  export function get_status(): string | number;
---

## WebAssemblyにおけるTypeScriptの重要性

RustをWasmにコンパイルすると、生のバウンダリは数値のみを扱います — `i32`、`f64`、そしてメモリオフセットです。TypeScriptはJavaScript側に**型安全な契約**を追加する手段を提供し、Wasmモジュールの利用者がオートコンプリート、コンパイル時のエラーチェック、自己文書化されたAPIを得られるようにします：

```
  TypeScriptバインディングなし         TypeScriptバインディングあり

  ┌──────────────────────┐           ┌──────────────────────┐
  │  JavaScript          │           │  TypeScript           │
  │                      │           │                      │
  │  wasm.greet(42)      │           │  wasm.greet(42)      │
  │  // エラーなし!      │           │  // TSエラー:        │
  │  // 実行時に         │           │  // 型 'number' を   │
  │  // クラッシュ       │           │  // 型 'string' に   │
  │                      │           │  // 割り当てることは  │
  │                      │           │  // できません        │
  └──────────────────────┘           └──────────────────────┘
```

型システムがバグをWasmバウンダリに到達する**前に**キャッチします。バウンダリでのデバッグは非常に困難です。

## wasm-bindgenによるTypeScript生成の仕組み

`wasm-pack build`を実行すると、ツールチェーンはいくつかのファイルを生成します：

```
  pkg/
  ├── my_crate_bg.wasm          ← コンパイル済みWasmバイナリ
  ├── my_crate_bg.wasm.d.ts     ← 生のWasmエクスポートの型
  ├── my_crate.js               ← JSグルーコード
  └── my_crate.d.ts             ← TypeScript定義 ★
```

`.d.ts`ファイルは`#[wasm_bindgen]`アノテーションから直接生成されます：

```rust
// Rust側
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[wasm_bindgen]
pub struct Counter {
    value: i32,
}

#[wasm_bindgen]
impl Counter {
    #[wasm_bindgen(constructor)]
    pub fn new(start: i32) -> Counter {
        Counter { value: start }
    }

    pub fn increment(&mut self) {
        self.value += 1;
    }

    pub fn get(&self) -> i32 {
        self.value
    }
}
```

生成結果：

```typescript
// 自動生成された my_crate.d.ts
export function greet(name: string): string;

export class Counter {
  free(): void;
  constructor(start: number);
  increment(): void;
  get(): number;
}
```

## 複雑な型のためのtsifyクレート

`#[wasm_bindgen]`は関数と不透明なクラスに対してうまく動作しますが、**プレーンなデータ型**にはクラスではなくTypeScriptインターフェースが必要です。そこで`tsify`の出番です：

```rust
use tsify::Tsify;
use serde::{Serialize, Deserialize};

#[derive(Tsify, Serialize, Deserialize)]
#[tsify(into_wasm_abi, from_wasm_abi)]
pub struct User {
    pub id: u32,
    pub name: String,
    pub email: Option<String>,
    pub roles: Vec<String>,
}

#[derive(Tsify, Serialize, Deserialize)]
#[tsify(into_wasm_abi, from_wasm_abi)]
pub enum Status {
    Loading,
    Ready { data: Vec<f64> },
    Error { message: String },
}
```

これにより、不透明なクラスの代わりにTypeScriptの**インターフェース**と**タグ付きユニオン**が生成されます：

```typescript
export interface User {
  id: number;
  name: string;
  email?: string;
  roles: string[];
}

export type Status =
  | { tag: "Loading" }
  | { tag: "Ready"; data: number[] }
  | { tag: "Error"; message: string };
```

### 構造体におけるtsify vs wasm_bindgen

| 機能                   | `#[wasm_bindgen]` struct   | `#[tsify]` struct           |
|------------------------|----------------------------|-----------------------------|
| TS出力                 | メソッド付き`class`         | `interface`（プレーンobj）   |
| 渡し方                 | ヒープポインタ（不透明）     | シリアライズされたJSON/obj    |
| 分割代入               | 不可                       | 可能（プレーンオブジェクト）   |
| パフォーマンス          | ゼロコピー（高速）          | シリアライゼーションのオーバーヘッド |
| 複雑なネスト            | 限定的                     | 完全サポート                 |
| 列挙型サポート          | フラットなCスタイルのみ      | タグ付きユニオン              |
| 使用場面               | 長寿命、メソッドあり         | データ転送オブジェクト         |

## typescript_custom_section属性

自動生成では不十分な場合、カスタムTypeScriptを直接注入できます：

```rust
#[wasm_bindgen(typescript_custom_section)]
const TS_APPEND: &str = r#"
export interface WasmMemoryStats {
    used_bytes: number;
    total_bytes: number;
    fragmentation_ratio: number;
}

export type EventCallback = (event: CustomEvent) => void;

export interface PluginApi {
    init(config: Record<string, unknown>): Promise<void>;
    process(data: Uint8Array): Promise<Uint8Array>;
    destroy(): void;
}
"#;
```

これは以下の場合に便利です：
- ブラウザAPIを参照する型（`HTMLCanvasElement`、`AudioContext`）
- コールバック関数のシグネチャ
- ジェネリックなラッパー型
- モジュール拡張

## RustからTypeScriptへの型マッピング

```
  Rust型                             TypeScript型
  ─────────────────────────────────  ────────────────────────
  i8, i16, i32                       number
  u8, u16, u32                       number
  f32, f64                           number
  i64, u64                           bigint
  i128, u128                         bigint
  bool                               boolean
  char                               string
  String, &str                       string
  Option<T>                          T | undefined
  Vec<T>                             T[]
  [T; N]                             T[]（固定サイズ情報は失われる）
  Box<[u8]>                          Uint8Array
  HashMap<String, V>                 Map<string, V>
  Result<T, E>                       T（Errは例外に変換）
  ()                                 void
  JsValue                            any ← これは避けましょう!
```

### JsValue（"any"エスケープハッチ）の回避

`JsValue`はTypeScriptの`any`にマッピングされ、型安全性の意味がなくなります。具体的な型を使いましょう：

```rust
// 悪い例: 型情報が失われる
#[wasm_bindgen]
pub fn process(input: JsValue) -> JsValue { /* ... */ }
// TS: process(input: any): any;

// 良い例: 完全に型付けされている
#[wasm_bindgen]
pub fn process(input: &str) -> String { /* ... */ }
// TS: process(input: string): string;

// 良い例: tsifyを使った複雑な型
#[wasm_bindgen]
pub fn process_user(user: User) -> ProcessResult { /* ... */ }
// TS: process_user(user: User): ProcessResult;
```

## ジェネリックラッパーパターン

一般的なパターンとして、Wasmモジュールに型付きラッパーを作成します：

```typescript
// wrapper.ts — 手書き、生成された.d.tsからインポート
import init, { greet, Counter, process_users } from './pkg/my_crate';
import type { User, Status, AppConfig } from './pkg/my_crate';

class WasmApi {
  private ready: Promise<void>;

  constructor() {
    this.ready = init();
  }

  async greet(name: string): Promise<string> {
    await this.ready;
    return greet(name);
  }

  async processUsers(users: User[], config: AppConfig): Promise<number> {
    await this.ready;
    return process_users(users, config);
  }

  createCounter(start: number): Counter {
    return new Counter(start);
  }
}

export const wasmApi = new WasmApi();
```

このパターンにより以下が得られます：
- 遅延初期化（初回使用時にWasmをロード）
- すべてのWasm機能への単一エントリポイント
- フレームワーク非依存（React、Vue、Svelteなどで動作）

## wasm-packビルドターゲットとTypeScript

```
  wasm-pack build --target ...

  ┌──────────┬───────────────────────────────────────┐
  │ ターゲット │ 出力                                  │
  ├──────────┼───────────────────────────────────────┤
  │ bundler  │ ESモジュール（.d.ts含む）               │
  │          │ webpack、vite、rollup向け               │
  ├──────────┼───────────────────────────────────────┤
  │ web      │ ESモジュール + 手動init()               │
  │          │ .d.ts含む、自分でinit()を呼ぶ           │
  ├──────────┼───────────────────────────────────────┤
  │ nodejs   │ CommonJSモジュール（.d.ts含む）         │
  │          │ require()またはimportでNode利用          │
  ├──────────┼───────────────────────────────────────┤
  │ no-modules│ グローバル変数、.d.tsなし              │
  │          │ レガシー、TypeScriptでは避ける           │
  └──────────┴───────────────────────────────────────┘
```

TypeScriptプロジェクトでは、常に`--target bundler`または`--target web`を使用してください。

## Wasm用tsconfig.jsonの設定

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "bundler",
    "strict": true,
    "types": ["@aspect/wasm-types"],
    "paths": {
      "my-wasm-pkg": ["./pkg/my_crate.d.ts"]
    }
  }
}
```

重要な設定：
- `"strict": true` — Wasmバウンダリでの型の不一致を検出
- `"target": "ES2020"` — `BigInt`サポート（`i64`/`u64`）に必要
- パスエイリアスにより、生成されたパッケージをクリーンにインポート可能

## まとめ

TypeScript統合は、生のWasmモジュールを開発者フレンドリーで型安全なライブラリに変換します。関数と不透明な型には`#[wasm_bindgen]`を、データ転送オブジェクトと列挙型には`tsify`を、エッジケースには`typescript_custom_section`を使用しましょう。バウンダリ全体で型情報を保持するために、常に`JsValue`よりも具体的なRust型を優先してください。生成された`.d.ts`ファイルにより、TypeScriptの利用者は手動の作業ゼロでオートコンプリート、ドキュメント、コンパイル時の安全性を得られます。
