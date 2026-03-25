---
title: "クロージャとコールバック：JSとRust間の連携"
slug: closures-callbacks
difficulty: intermediate
tags: [getting-started]
order: 29
description: Wasm境界を越えたRustクロージャの仕組みを学ぶ — Closure::wrap、Closure::once、'staticライフタイム要件、JSとRust間の関数受け渡し、forget()とinto_js_value()によるクロージャのメモリ管理。
starter_code: |
  fn main() {
      println!("=== Rustのクロージャとコールバック ===\n");

      // --- 基本的なクロージャ構文 ---
      let greet = |name: &str| format!("Hello, {}!", name);
      println!("基本クロージャ: {}", greet("Wasm"));

      // --- 変数をキャプチャするクロージャ ---
      let multiplier = 3;
      let multiply = |x: i32| x * multiplier;
      println!("参照キャプチャ: 5 * {} = {}", multiplier, multiply(5));

      // --- moveクロージャ（所有権の移動） ---
      let prefix = String::from("LOG");
      let logger = move |msg: &str| format!("[{}] {}", prefix, msg);
      // prefixはクロージャに移動済み
      println!("moveクロージャ: {}", logger("system started"));

      // --- FnOnce: 一度だけ呼び出し可能 ---
      let data = vec![1, 2, 3, 4, 5];
      let consume = move || {
          let sum: i32 = data.iter().sum();
          println!("FnOnceがvecを消費、合計 = {}", sum);
          data  // dataの所有権を返す
      };
      let returned = consume();
      // consume()は再呼び出し不可 — dataが移動された
      println!("返されたデータ: {:?}", returned);

      // --- FnMut: キャプチャした状態を変更可能 ---
      let mut counter = 0;
      let mut increment = || {
          counter += 1;
          counter
      };
      println!("\nFnMutカウンター:");
      for _ in 0..5 {
          print!("  count = {}", increment());
          println!();
      }

      // --- Fn: 不変借用、何度でも呼び出し可能 ---
      let threshold = 10;
      let check = |val: i32| -> bool { val > threshold };
      println!("\nFn check(5 > {}): {}", threshold, check(5));
      println!("Fn check(15 > {}): {}", threshold, check(15));

      // --- コールバック登録のシミュレーション ---
      println!("\n--- コールバックシミュレーション ---");

      fn register_callback<F: Fn(i32) -> i32>(name: &str, callback: F) {
          let result = callback(42);
          println!("コールバック '{}' に入力42 => {}", name, result);
      }

      register_callback("double", |x| x * 2);
      register_callback("square", |x| x * x);
      register_callback("negate", |x| -x);

      // --- Boxedコールバック（ヒープ割り当て、Closure::wrapに相当） ---
      println!("\n--- Boxedコールバック（Closure::wrapのシミュレーション） ---");

      let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
          Box::new(|x| x + 1),
          Box::new(|x| x * 10),
          Box::new(|x| x - 5),
      ];

      for (i, cb) in callbacks.iter().enumerate() {
          println!("  boxed_callback[{}](100) = {}", i, cb(100));
      }

      // --- 'staticライフタイム要件のシミュレーション ---
      println!("\n--- 'staticライフタイムデモ ---");

      fn needs_static<F: Fn() -> String + 'static>(f: F) -> String {
          f()
      }

      // これは動作する: 文字列リテラルは'static
      let result = needs_static(|| String::from("I am 'static!"));
      println!("  'staticクロージャの結果: {}", result);

      // これも動作する: moveが所有データをキャプチャ
      let owned = String::from("owned data");
      let result = needs_static(move || format!("キャプチャ: {}", owned));
      println!("  move + 'staticの結果: {}", result);

      // --- forget()とdrop動作のシミュレーション ---
      println!("\n--- メモリ管理シミュレーション ---");

      struct ClosureWrapper {
          name: String,
      }

      impl ClosureWrapper {
          fn new(name: &str) -> Self {
              println!("  クロージャ '{}' を作成", name);
              ClosureWrapper { name: name.to_string() }
          }

          fn forget(self) {
              println!("  '{}' を忘却 — メモリリーク（dropされない）", self.name);
              std::mem::forget(self);
          }
      }

      impl Drop for ClosureWrapper {
          fn drop(&mut self) {
              println!("  クロージャ '{}' をdrop", self.name);
          }
      }

      let temp = ClosureWrapper::new("temporary");
      drop(temp);  // 明示的なdrop

      let leaked = ClosureWrapper::new("event_listener");
      leaked.forget();  // Closure::forget()のシミュレーション

      let scoped = ClosureWrapper::new("scoped");
      // scopedはブロックの終わりで自動的にdropされる
      println!("  (scopedはブロックの終わりでdropされます)");
  }
expected_output: |
  === Rustのクロージャとコールバック ===

  基本クロージャ: Hello, Wasm!
  参照キャプチャ: 5 * 3 = 15
  moveクロージャ: [LOG] system started
  FnOnceがvecを消費、合計 = 15
  返されたデータ: [1, 2, 3, 4, 5]

  FnMutカウンター:
    count = 1
    count = 2
    count = 3
    count = 4
    count = 5

  Fn check(5 > 10): false
  Fn check(15 > 10): true

  --- コールバックシミュレーション ---
  コールバック 'double' に入力42 => 84
  コールバック 'square' に入力42 => 1764
  コールバック 'negate' に入力42 => -42

  --- Boxedコールバック（Closure::wrapのシミュレーション） ---
    boxed_callback[0](100) = 101
    boxed_callback[1](100) = 1000
    boxed_callback[2](100) = 95

  --- 'staticライフタイムデモ ---
    'staticクロージャの結果: I am 'static!
    move + 'staticの結果: キャプチャ: owned data

  --- メモリ管理シミュレーション ---
    クロージャ 'temporary' を作成
    クロージャ 'temporary' をdrop
    クロージャ 'event_listener' を作成
    'event_listener' を忘却 — メモリリーク（dropされない）
    クロージャ 'scoped' を作成
    (scopedはブロックの終わりでdropされます)
    クロージャ 'scoped' をdrop
---

## クロージャとは？

**クロージャ**とは、周囲のスコープから変数をキャプチャできる無名関数です。Rustでは、クロージャは最も強力な機能の一つであり、JavaScriptがコールバックに大きく依存しているため、Wasmインターオペレーションには不可欠です。

```rust
// クロージャの3つの等価な書き方:
let add = |a, b| a + b;           // 型推論
let add = |a: i32, b: i32| a + b; // 明示的な型
let add = |a: i32, b: i32| -> i32 { a + b }; // 完全な構文
```

## 3つのクロージャトレイト

Rustは、キャプチャした変数の使い方に基づいてクロージャを3つのトレイトに分類します：

```
+----------+-------------------+---------------------------+
|  トレイト |  キャプチャ方法    |  呼び出し回数             |
+----------+-------------------+---------------------------+
|  Fn      |  &T（不変借用）    |  何度でも                 |
|  FnMut   |  &mut T（可変借用）|  何度でも（&mutが必要）    |
|  FnOnce  |  T（値渡し）       |  1回のみ                  |
+----------+-------------------+---------------------------+
```

コンパイラは、キャプチャした変数に対する操作に基づいて、クロージャがどのトレイトを実装するかを自動的に判断します：

```rust
let x = 5;
let reads_x = || println!("{}", x);    // Fn（xを借用）
let mut y = 5;
let mutates_y = || { y += 1; };        // FnMut（yを可変借用）
let z = String::from("hello");
let consumes_z = || { drop(z); };      // FnOnce（zを移動）
```

**重要：** すべての`FnOnce`が`FnMut`とは限らず、すべての`FnMut`が`Fn`とは限りません。しかし `Fn` ⊂ `FnMut` ⊂ `FnOnce` です（`Fn`を実装するクロージャは`FnMut`と`FnOnce`も実装します）。

## `move`キーワード

デフォルトでは、クロージャは必要最小限の借用で変数をキャプチャします。`move`キーワードはクロージャにキャプチャしたすべての変数の**所有権**を強制的に取得させます：

```rust
let name = String::from("Alice");
let greet = move || println!("Hello, {}", name);
// nameはクロージャに移動したため、ここではアクセス不可
greet();
```

これはWasmにとって重要です。JavaScriptに渡されるクロージャは`'static`でなければならず、Rustスタックからの借用はできません。JSがクロージャを呼び出す時点でスタックフレームが消えている可能性があるためです。

## wasm-bindgenにおけるクロージャ

Wasmの世界では、`wasm_bindgen::closure::Closure`がRustのクロージャをラップして、JavaScriptから呼び出せるようにします。主に2つのコンストラクタがあります：

### Closure::wrap

```rust
// Closure::wrapは長寿命のJS呼び出し可能クロージャを作成する
let cb = Closure::wrap(Box::new(|event: web_sys::MouseEvent| {
    // クリック処理
}) as Box<dyn FnMut(web_sys::MouseEvent)>);

element.add_event_listener_with_callback("click", cb.as_ref().unchecked_ref())?;
```

### Closure::once

```rust
// Closure::onceはワンショットクロージャ（FnOnce）を作成する
let cb = Closure::once(move || {
    // これは一度だけ実行され、その後クロージャはクリーンアップされる
    web_sys::console::log_1(&"Loaded!".into());
});
```

## `'static`ライフタイム要件

RustからJavaScriptにクロージャを渡す場合、`'static`ライフタイム境界を満たす必要があります。これは以下を意味します：

```
┌─────────────────────────────────────────────────────┐
│                Rustスタックフレーム                    │
│                                                      │
│  let local_string = String::from("hello");           │
│                                                      │
│  // コンパイル不可 — クロージャがlocal_stringを借用   │
│  let bad = Closure::wrap(Box::new(|| {               │
│      console::log_1(&local_string.into());           │
│  }) as Box<dyn Fn()>);                               │
│                                                      │
│  // 動作する — moveがクロージャに所有権を与える       │
│  let good = Closure::wrap(Box::new(move || {         │
│      console::log_1(&local_string.into());           │
│  }) as Box<dyn FnOnce()>);                           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

理由：Rust関数が返ると、そのスタックは消えます。後でJSがクロージャを呼び出すと、借用された参照はダングリングになります。`'static`境界はクロージャが必要なものをすべて所有していることを保証します。

## メモリ管理：forget() vs into_js_value()

これはWasmクロージャの最も難しい部分の一つです。Rustで`Closure`がdropされると、JSコールバックは無効になります。しかし、コールバックが作成元のRust関数より長く生きる必要があることがよくあります。

### closure.forget()

```rust
let cb = Closure::wrap(Box::new(|| { /* ... */ }) as Box<dyn Fn()>);
element.set_onclick(Some(cb.as_ref().unchecked_ref()));
cb.forget();  // メモリリーク — クロージャは永久に生き続ける
```

**利点：** シンプルで、常に動作する。
**欠点：** メモリリーク。イベントリスナーを頻繁に追加/削除すると、メモリが際限なく増加する。

### closure.into_js_value()

```rust
let cb = Closure::wrap(Box::new(|| { /* ... */ }) as Box<dyn Fn()>);
let js_func: JsValue = cb.into_js_value();
// js_funcがクロージャを所有 — JSのGCが回収するとき解放される
```

**利点：** メモリリークなし — JSのGCがライフタイムを管理する。
**欠点：** Rustの`Closure`ハンドルを失う（後でリスナーを簡単に削除できない）。

### 比較表

| メソッド | メモリリーク？ | JS GCが管理？ | ユースケース |
|--------|-------------|----------------|----------|
| `forget()` | あり | いいえ | 長寿命イベントリスナー |
| `into_js_value()` | なし | はい | 渡して忘れるコールバック |
| 構造体に保存 | なし | いいえ | 制御されたライフタイム（最善） |
| `Closure::once` | なし | 自動クリーンアップ | ワンショットコールバック |

## ベストプラクティス：構造体にクロージャを保存する

最もクリーンなアプローチは、コールバックが必要な間生き続けるRust構造体に`Closure`を保存することです：

```rust
#[wasm_bindgen]
pub struct App {
    click_handler: Option<Closure<dyn FnMut(MouseEvent)>>,
}

#[wasm_bindgen]
impl App {
    pub fn new() -> App {
        App { click_handler: None }
    }

    pub fn setup_click(&mut self, element: &HtmlElement) {
        let cb = Closure::wrap(Box::new(|e: MouseEvent| {
            // クリック処理
        }) as Box<dyn FnMut(MouseEvent)>);

        element.set_onclick(Some(cb.as_ref().unchecked_ref()));
        self.click_handler = Some(cb);  // 保存 — リークしない
    }
}
// Appがdropされると、click_handlerもdropされ、コールバックは無効になる
```

## JavaScriptからRustへの関数受け渡し

JS関数を引数として受け取ることもできます：

```rust
#[wasm_bindgen]
pub fn call_js_function(f: &js_sys::Function) {
    let this = JsValue::NULL;
    let arg = JsValue::from(42);
    f.call1(&this, &arg).unwrap();
}
```

JavaScript側：
```js
import { call_js_function } from './pkg/my_module.js';
call_js_function((x) => console.log("Got from Rust:", x));
// 出力: "Got from Rust: 42"
```

## よくあるパターン

### デバウンスコールバック

```rust
use std::cell::RefCell;
use std::rc::Rc;

let timeout_id = Rc::new(RefCell::new(None));
let tid = timeout_id.clone();

let debounced = Closure::wrap(Box::new(move || {
    if let Some(id) = tid.borrow_mut().take() {
        window.clear_timeout_with_handle(id);
    }
    let new_id = window.set_timeout_with_callback_and_timeout_and_arguments_0(
        actual_handler.as_ref().unchecked_ref(), 300
    ).unwrap();
    *tid.borrow_mut() = Some(new_id);
}) as Box<dyn FnMut()>);
```

### requestAnimationFrameループ

```rust
fn request_animation_frame(f: &Closure<dyn FnMut()>) {
    window().unwrap()
        .request_animation_frame(f.as_ref().unchecked_ref())
        .expect("should register `requestAnimationFrame`");
}

let f = Rc::new(RefCell::new(None));
let g = f.clone();

*g.borrow_mut() = Some(Closure::wrap(Box::new(move || {
    // ここでフレームをレンダリング
    request_animation_frame(f.borrow().as_ref().unwrap());
}) as Box<dyn FnMut()>));

request_animation_frame(g.borrow().as_ref().unwrap());
```

## まとめ

1. **Rustクロージャ**は、変数のキャプチャ方法に応じて`Fn`、`FnMut`、`FnOnce`を実装する
2. **`move`クロージャ**は`'static`要件のため、Wasmではほぼ常に必要
3. **`Closure::wrap`**は複数回使用可能なJSコールバックを作成し、**`Closure::once`**は単発のコールバックを作成する
4. **`forget()`**はメモリリークするがシンプル、**`into_js_value()`**はJS GCにライフタイム管理を委ねる
5. **ベストプラクティス：**`Closure`の値を構造体フィールドに保存し、ライフタイムを明示的に管理する
6. **JS → Rust：**`&js_sys::Function`を受け取ることでRustでJavaScript関数を受け取れる
