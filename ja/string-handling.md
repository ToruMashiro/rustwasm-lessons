---
title: 文字列処理の詳細
slug: string-handling
difficulty: intermediate
tags: [data-structures]
order: 22
description: RustとJavaScript間の文字列受け渡しをマスターする — UTF-8とUTF-16エンコーディング、パフォーマンスコスト、最適化戦略。
starter_code: |
  fn main() {
      // 文字列エンコーディングのデモ

      // 1. UTF-8エンコーディング（Rustが使用するもの）
      let text = "Hello, 世界! 🌍";
      println!("Text: {}", text);
      println!("Byte length (UTF-8): {}", text.len());
      println!("Char count: {}", text.chars().count());

      // 2. 個々のバイト値
      let simple = "ABC";
      let bytes: Vec<u8> = simple.bytes().collect();
      println!("\n\"ABC\" as UTF-8 bytes: {:?}", bytes);

      // 3. マルチバイト文字
      let japanese = "日本語";
      println!("\n\"日本語\" UTF-8 bytes: {} bytes", japanese.len());
      println!("\"日本語\" chars: {} characters", japanese.chars().count());

      // 4. 絵文字（UTF-8で4バイト）
      let emoji = "🦀";
      println!("\n\"🦀\" UTF-8 bytes: {} bytes", emoji.len());

      // 5. 文字列の構築（Wasmでよく使うパターン）
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

## エンコーディングの不一致

これはWasmで最もよくあるパフォーマンスの落とし穴です:

| | Rust / Wasm | JavaScript |
|---|---|---|
| **エンコーディング** | UTF-8 | UTF-16 |
| **ASCII文字あたりのバイト数** | 1 | 2 |
| **日本語文字あたりのバイト数** | 3 | 2 |
| **絵文字あたりのバイト数** | 4 | 4 |

文字列がWasm境界を越えるたびに、**再エンコード**が必要です:

```
JS文字列 (UTF-16) → UTF-8にエンコード → Wasmメモリにコピー → Rust &str
Rust String (UTF-8) → UTF-8からデコード → JS文字列を生成 (UTF-16)
```

このコピーは**毎回**発生します — 現在のWasmでは回避できません。

## wasm-bindgenによる文字列処理

### JS → Rust (`&str`)

```rust
#[wasm_bindgen]
pub fn process(input: &str) -> String {
    // inputはすでにWasmメモリ内のUTF-8
    // wasm-bindgenが呼び出し前にJS UTF-16からエンコード済み
    format!("Processed: {}", input)
    // String返却 → wasm-bindgenがUTF-8をデコード → JSがUTF-16文字列を取得
}
```

内部の動作:
1. JSが `TextEncoder.encode()` を呼んでUTF-16 → UTF-8に変換
2. UTF-8バイトをWasmリニアメモリにコピー
3. ポインタ + 長さをRustに渡す
4. Rustは有効な `&str` として参照する

### Rust → JS (`String`)

1. RustがUTF-8バイトをリニアメモリに書き込む
2. wasm-bindgenが `TextDecoder.decode()` を呼んでUTF-8 → UTF-16に変換
3. JSがネイティブ文字列を取得する

## パフォーマンスコスト

```
操作                       時間 (1MB文字列)
─────────────────────────────────────────
JS → Rust (&str)          ~2ms (エンコード + コピー)
Rust → JS (String)        ~2ms (デコード + コピー)
Rust内部 (コピーなし)      ~0ms (ポインタのみ)
```

**ルール**: 文字列の境界越えを最小限にする。すべての文字列処理を片側で行う。

## 最適化戦略

### 1. バッチ処理 — アイテムごとに境界を越えない

```rust
// 悪い例: N回境界を越える
#[wasm_bindgen]
pub fn process_one(item: &str) -> String { /* ... */ }

// 良い例: 1回だけ境界を越える
#[wasm_bindgen]
pub fn process_all(items_json: &str) -> String {
    // すべてのアイテムをパース、処理、全結果を返す
    // 境界越えは合計2回のみ
}
```

### 2. 文字列の代わりに数値IDを使う

```rust
// 悪い例: 文字列をやり取りする
pub fn get_user_name(name: &str) -> String { /* ... */ }

// 良い例: IDを渡し、文字列はRust内に保持
pub fn create_user(name: &str) -> u32 { /* IDを返す */ }
pub fn get_user_name(id: u32) -> String { /* IDで検索 */ }
```

### 3. バイナリデータには &[u8] を使う

```rust
// バイナリデータを文字列としてエンコードしない
pub fn process_base64(data: &str) -> String { /* ... */ }

// 生バイトを渡す
pub fn process_bytes(data: &[u8]) -> Vec<u8> { /* ... */ }
```

### 4. キャパシティを事前確保する

```rust
// 悪い例: 多数の小さなアロケーション
let mut result = String::new();
for item in items {
    result.push_str(&format!("{},", item));
}

// 良い例: 1回のアロケーション
let mut result = String::with_capacity(items.len() * 20);
for item in items {
    result.push_str(&format!("{},", item));
}
```

## Wasmでよくある文字列パターン

### CSV/JSON処理

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

### テンプレートレンダリング

```rust
#[wasm_bindgen]
pub fn render_template(template: &str, name: &str, count: u32) -> String {
    template
        .replace("{{name}}", name)
        .replace("{{count}}", &count.to_string())
}
```

## 試してみよう

**Run** をクリックして、UTF-8エンコーディングの動作を確認しましょう — バイト長、マルチバイト文字、文字列の構築を体験できます。日本語文字がUTF-8では3バイトですが、JavaScriptのUTF-16では2バイトであることに注目してください。
