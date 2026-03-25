---
title: Wasmのデバッグ
slug: debugging
difficulty: beginner
tags: [getting-started]
order: 10
description: コンソールログ、パニックフック、ブラウザDevTools、エラーハンドリングのベストプラクティスを使ってRust/Wasmアプリケーションをデバッグします。
starter_code: |
  fn main() {
      // 一般的なデバッグシナリオのシミュレーション

      // 1. 基本的なログ出力
      println!("Debug: starting application");

      // 2. 変数の検査
      let data = vec![1, 2, 3, 4, 5];
      println!("Debug: data = {:?}", data);
      println!("Debug: data length = {}", data.len());

      // 3. 条件付きデバッグ
      let value = 42;
      if value > 40 {
          println!("Warning: value {} exceeds threshold", value);
      }

      // 4. エラーのシミュレーション
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

## Wasmのデバッグが異なる理由

RustがWasm内でパニックすると、ブラウザのコンソールには役に立たない`unreachable`エラーが表示されます。スタックトレースもエラーメッセージもありません。このレッスンでは、その問題を解決する方法を紹介します。

## ステップ1：より良いパニックメッセージ

`console_error_panic_hook`を追加して、読みやすいパニックメッセージを取得します：

```toml
[dependencies]
console_error_panic_hook = "0.1"
wasm-bindgen = "0.2"
```

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn init() {
    // 起動時に一度だけ呼び出す
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn might_panic(x: i32) -> i32 {
    // フックなし: "unreachable"
    // フックあり: "panicked at 'index out of bounds: len is 3, index is 5'"
    let v = vec![1, 2, 3];
    v[x as usize]
}
```

## ステップ2：Rustからのコンソールログ

### シンプルな方法：`web_sys::console`を使用

```rust
use web_sys::console;

// 文字列をログ出力
console::log_1(&"Hello from Rust!".into());

// フォーマット付きログ
console::log_2(&"Value:".into(), &JsValue::from(42));

// 警告とエラー
console::warn_1(&"This is a warning".into());
console::error_1(&"This is an error".into());
```

### よりクリーンな方法：ログマクロを作成

```rust
macro_rules! log {
    ($($t:tt)*) => {
        web_sys::console::log_1(
            &wasm_bindgen::JsValue::from_str(&format!($($t)*))
        );
    };
}

// 使用例:
log!("User {} logged in at {}", name, timestamp);
log!("Data: {:?}", my_vec);
```

## ステップ3：ブラウザDevTools

### ソースマップ

デバッグ情報付きでビルドすると、DevToolsでRustのソースコードを確認できます：

```bash
# デバッグビルド（バイナリが大きくなるが、デバッグ情報あり）
wasm-pack build --dev --target web

# リリースビルド（デバッグ情報なし）
wasm-pack build --target web
```

### Networkタブ

`.wasm`ファイルが正しく読み込まれているか確認します：
- ステータスが200であること
- Content-Typeが`application/wasm`であること
- サイズで最適化が効いているか確認

### Performanceタブ

Wasmの実行をプロファイリングします：
1. DevTools → Performanceを開く
2. Recordをクリック
3. アプリを操作する
4. 記録を停止
5. フレームチャートで"wasm-function"エントリを探す

## ステップ4：エラーハンドリングパターン

### パニックの代わりにResultを返す

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

JavaScript側では、try/catchになります：

```js
try {
    const result = safe_divide(10, 0);
} catch (e) {
    console.error("Wasm error:", e); // "Division by zero"
}
```

### カスタムエラー型

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

## よくあるデバッグチェックリスト

| 問題 | 原因 | 解決方法 |
|------|------|---------|
| コンソールに`unreachable` | フックなしのRustパニック | `console_error_panic_hook`を追加 |
| `.wasm`ファイルが読み込めない | パスまたはMIMEタイプが間違い | Networkタブを確認し、正しいContent-Typeで配信 |
| 読み込み時に`LinkError` | インポートが不足 | すべての`extern "C"`インポートが満たされているか確認 |
| `RuntimeError: memory` | Wasmメモリ不足 | メモリを増やすかメモリリークを修正 |
| 関数が見つからない | エクスポートされていない | `#[wasm_bindgen]`と`pub`を追加 |
| 型が一致しない | wasm-bindgenのバージョン不一致 | JSグルーと.wasmが同じビルドのものか確認 |

## 試してみよう

**Run**をクリックして、デバッグ出力の例を確認しましょう。実際のWasmプロジェクトでは、`println!`を`console::log_1`に置き換えてください。
