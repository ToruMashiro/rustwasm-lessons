---
title: Fetch APIコール
slug: fetch-api
difficulty: intermediate
tags: [api]
order: 6
description: ブラウザのFetch APIとasync/awaitを使って、Rust/WasmからHTTPリクエストを行う方法を学びます。
starter_code: |
  use wasm_bindgen::prelude::*;
  use wasm_bindgen_futures::JsFuture;
  use web_sys::{Request, RequestInit, Response, window};

  #[wasm_bindgen]
  pub async fn fetch_json(url: &str) -> Result<JsValue, JsValue> {
      // リクエストを設定
      let mut opts = RequestInit::new();
      opts.method("GET");

      let request = Request::new_with_str_and_init(url, &opts)?;
      request.headers()
          .set("Accept", "application/json")?;

      // リクエストを送信
      let window = window().unwrap();
      let resp_value = JsFuture::from(
          window.fetch_with_request(&request)
      ).await?;

      // レスポンスを解析
      let resp: Response = resp_value.dyn_into()?;

      if !resp.ok() {
          return Err(JsValue::from_str(
              &format!("HTTPエラー: {}", resp.status())
          ));
      }

      let json = JsFuture::from(resp.json()?).await?;
      Ok(json)
  }
expected_output: |
  https://api.example.com/data を取得中...
  ステータス: 200 OK
  レスポンス: { "status": "ok", "data": [...] }
---

## はじめに

Rust/Wasmは `web-sys` を通じてブラウザのネイティブFetch APIを使ったHTTPリクエストが可能です。`wasm-bindgen-futures` と組み合わせることで、完全なasync/awaitサポートが得られます — RustのFutureはJavaScriptのPromiseに自動的にブリッジされます。

## 必要な依存関係

```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Request",
    "RequestInit",
    "RequestMode",
    "Response",
    "Headers",
]
```

## Wasmでの非同期処理の仕組み

Wasmはシングルスレッドで、ブラウザのメインスレッド上で動作します。Wasmでの非同期処理は以下のように機能します：

1. `wasm-bindgen-futures` がRustの `Future` とJSの `Promise` を相互変換
2. `.await` した時点で制御がJSイベントループに戻る
3. Promiseが解決されると、Rustコードが再開される

```rust
// このRust非同期関数は...
#[wasm_bindgen]
pub async fn do_work() -> Result<JsValue, JsValue> {
    let result = some_async_operation().await?;
    Ok(result)
}

// ...JavaScriptではこうなる：
// const result = await do_work();  // Promiseを返す
```

## GETリクエスト

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response, window};

#[wasm_bindgen]
pub async fn fetch_json(url: &str) -> Result<JsValue, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");

    let request = Request::new_with_str_and_init(url, &opts)?;
    let window = window().unwrap();
    let resp: Response = JsFuture::from(
        window.fetch_with_request(&request)
    ).await?.dyn_into()?;

    let json = JsFuture::from(resp.json()?).await?;
    Ok(json)
}
```

## JSONボディ付きPOSTリクエスト

```rust
use js_sys::JSON;

#[wasm_bindgen]
pub async fn post_json(url: &str, body: JsValue) -> Result<JsValue, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("POST");
    opts.body(Some(&JSON::stringify(&body)?));

    let request = Request::new_with_str_and_init(url, &opts)?;
    request.headers().set("Content-Type", "application/json")?;

    let window = window().unwrap();
    let resp: Response = JsFuture::from(
        window.fetch_with_request(&request)
    ).await?.dyn_into()?;

    let json = JsFuture::from(resp.json()?).await?;
    Ok(json)
}
```

## JavaScriptからの呼び出し

```js
import init, { fetch_json, post_json } from './pkg/my_wasm.js';

await init();

// GET
const data = await fetch_json("https://api.example.com/items");
console.log(data);

// POST
const result = await post_json("https://api.example.com/items", {
    name: "New Item",
    value: 42
});
```

## エラーハンドリング

`resp.ok()` を必ず確認しましょう — Fetch APIはHTTPエラー（4xx/5xx）でもスローしません：

```rust
let resp: Response = /* ... */;

if !resp.ok() {
    let status = resp.status();
    let text = JsFuture::from(resp.text()?).await?;
    return Err(JsValue::from_str(
        &format!("HTTP {}: {}", status, text.as_string().unwrap_or_default())
    ));
}
```

## セキュリティ：CORS

Wasmからのすべてのfetchリクエストは、ブラウザの**CORS**（Cross-Origin Resource Sharing）ポリシーに従います：

- **同一オリジン**のリクエストは常に成功
- **クロスオリジン**のリクエストはサーバーが `Access-Control-Allow-Origin` ヘッダーを送信する必要がある
- リクエストモードを明示的に設定できます：

```rust
opts.mode(web_sys::RequestMode::Cors);  // デフォルト
opts.mode(web_sys::RequestMode::NoCors); // 不透明なレスポンス
```

Wasmモジュールはホスティングページと**同じオリジン制限**で動作します — 特別な権限はありません。

## 試してみよう

JSONリクエストボディ付きのPOSTサポートを追加してみましょう。
