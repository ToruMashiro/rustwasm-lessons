---
title: DOM操作
slug: dom-manipulation
difficulty: beginner
tags: [dom, getting-started]
order: 5
description: web-sysバインディングを使って、RustからブラウザのDOMに直接アクセス・操作する方法を学びます。
starter_code: |
  use wasm_bindgen::prelude::*;
  use web_sys::{Document, Element, window};

  #[wasm_bindgen]
  pub fn create_element(tag: &str, text: &str) -> Result<Element, JsValue> {
      let document: Document = window()
          .unwrap()
          .document()
          .unwrap();

      let element = document.create_element(tag)?;
      element.set_text_content(Some(text));
      element.set_attribute("class", "wasm-created")?;

      document.body()
          .unwrap()
          .append_child(&element)?;

      Ok(element)
  }
expected_output: |
  <div> 要素を作成しました。テキスト: "Rustからこんにちは！"
  要素がドキュメントのbodyに追加されました
---

## はじめに

`web-sys` クレートは**すべての**Web APIに対するRustバインディングを提供します — DOM、Canvas、WebGL、Fetch、WebSocketなど。これらのバインディングはWebIDL仕様から自動生成されているため、ブラウザAPIと正確に一致します。

## web-sysの仕組み

`web-sys` はJavaScript Web APIの薄いラッパーです。各APIはCargoフィーチャーとして有効にする必要があります：

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
    "console",
]
```

リストしたフィーチャーだけがコンパイルされるため、バイナリサイズを小さく保てます。

## DOMへのアクセス

エントリポイントはJavaScriptと全く同じです：

```rust
use web_sys::window;

// window → document → body（JSと同じチェーン）
let window = window().expect("グローバルwindowがありません");
let document = window.document().expect("documentがありません");
let body = document.body().expect("bodyがありません");
```

注意：`window()`、`document()`、`body()` は `Option` を返します — 非ブラウザ環境（jsdomなしのNode.jsなど）で呼ばれた場合に失敗する可能性があります。

## 要素の作成と変更

```rust
// 新しい要素を作成
let div = document.create_element("div")?;

// コンテンツと属性を設定
div.set_text_content(Some("Rustからこんにちは！"));
div.set_attribute("id", "my-div")?;
div.set_attribute("class", "highlight")?;

// inner HTMLを設定（ユーザー入力のXSSリスクに注意）
div.set_inner_html("<strong>太字テキスト</strong>");

// DOMに追加
body.append_child(&div)?;
```

## 要素の検索

```rust
// IDで検索
let el = document.get_element_by_id("my-div");

// CSSセレクタで検索（最初にマッチした要素を返す）
let el = document.query_selector(".highlight")?;

// すべてのマッチを取得
let list = document.query_selector_all("div.item")?;
for i in 0..list.length() {
    let node = list.get(i).unwrap();
    // ... 各ノードを処理
}
```

## イベントリスナーの追加

`Closure::wrap` を使ってRustクロージャをJavaScriptコールバックに変換します：

```rust
use wasm_bindgen::closure::Closure;
use web_sys::HtmlElement;

let button: HtmlElement = document
    .get_element_by_id("my-btn")
    .unwrap()
    .dyn_into()?;

let closure = Closure::wrap(Box::new(move |_event: web_sys::MouseEvent| {
    web_sys::console::log_1(&"クリックされました！".into());
}) as Box<dyn FnMut(_)>);

button.add_event_listener_with_callback(
    "click",
    closure.as_ref().unchecked_ref(),
)?;

// 重要：Rustがクロージャを解放するのを防ぐ
// （そうしないとコールバックが無効になる）
closure.forget();
```

**なぜ `closure.forget()` が必要？** Rustは通常、値がスコープ外になると解放します。しかしこのクロージャはブラウザのイベントシステムに所有されています。`.forget()` を呼ぶことで、Rustにメモリを解放しないよう伝えます。長寿命のコールバックにはこれで問題ありません — 短寿命のものは、`Closure` を構造体に保持してライフタイムを制御しましょう。

## よく使うweb-sysフィーチャー

| フィーチャー | 提供する機能 | ユースケース |
|-------------|-------------|-------------|
| `"Window"` | `window()`、タイマー、location | エントリポイント |
| `"Document"` | DOM検索、要素作成 | DOM操作 |
| `"Element"` | 属性、クラス、子要素 | 要素の変更 |
| `"HtmlElement"` | スタイル、クリック、フォーカス | インタラクティブ要素 |
| `"HtmlCanvasElement"` | Canvas 2D/WebGL | グラフィックス |
| `"console"` | `console::log_1()` | デバッグ |
| `"Storage"` | `local_storage()` | データの永続化 |
| `"MouseEvent"` | クリック座標、ボタン | イベント処理 |
| `"KeyboardEvent"` | キーコード、修飾キー | キーボード入力 |
| `"RequestInit"` | Fetchオプション | HTTPリクエスト |

## エラーハンドリングパターン

ほとんどのweb-sysメソッドは `Result<T, JsValue>` を返します。`?` を使ってエラーを伝播しましょう：

```rust
#[wasm_bindgen]
pub fn setup() -> Result<(), JsValue> {
    let document = window().unwrap().document().unwrap();
    let canvas = document.create_element("canvas")?;
    canvas.set_attribute("width", "800")?;
    canvas.set_attribute("height", "600")?;
    document.body().unwrap().append_child(&canvas)?;
    Ok(())
}
```

## 試してみよう

`<button>` 要素を作成し、`Closure::wrap` を使ってクリックイベントリスナーを追加してみましょう。
