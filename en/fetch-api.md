---
title: Fetch API Calls
slug: fetch-api
difficulty: intermediate
tags: [api]
order: 6
description: Make HTTP requests from Rust/Wasm using the browser's Fetch API with async/await.
starter_code: |
  use wasm_bindgen::prelude::*;
  use wasm_bindgen_futures::JsFuture;
  use web_sys::{Request, RequestInit, Response, window};

  #[wasm_bindgen]
  pub async fn fetch_json(url: &str) -> Result<JsValue, JsValue> {
      // Configure the request
      let mut opts = RequestInit::new();
      opts.method("GET");

      let request = Request::new_with_str_and_init(url, &opts)?;
      request.headers()
          .set("Accept", "application/json")?;

      // Send the request
      let window = window().unwrap();
      let resp_value = JsFuture::from(
          window.fetch_with_request(&request)
      ).await?;

      // Parse the response
      let resp: Response = resp_value.dyn_into()?;

      if !resp.ok() {
          return Err(JsValue::from_str(
              &format!("HTTP error: {}", resp.status())
          ));
      }

      let json = JsFuture::from(resp.json()?).await?;
      Ok(json)
  }
expected_output: |
  Fetching https://api.example.com/data...
  Status: 200 OK
  Response: { "status": "ok", "data": [...] }
---

## Introduction

Rust/Wasm can make HTTP requests using the browser's native Fetch API through `web-sys`. Combined with `wasm-bindgen-futures`, you get full async/await support — Rust futures are bridged to JavaScript Promises automatically.

## Required dependencies

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

## How async works in Wasm

Wasm is single-threaded and runs on the browser's main thread. Async in Wasm works by:

1. `wasm-bindgen-futures` converts Rust `Future` ↔ JS `Promise`
2. When you `.await`, control returns to the JS event loop
3. When the Promise resolves, your Rust code resumes

```rust
// This Rust async function...
#[wasm_bindgen]
pub async fn do_work() -> Result<JsValue, JsValue> {
    let result = some_async_operation().await?;
    Ok(result)
}

// ...becomes this in JavaScript:
// const result = await do_work();  // returns a Promise
```

## GET request

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

## POST request with JSON body

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

## Calling from JavaScript

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

## Error handling

Always check `resp.ok()` — the Fetch API doesn't throw on HTTP errors (4xx/5xx):

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

## Security: CORS

All fetch requests from Wasm follow the browser's **CORS** (Cross-Origin Resource Sharing) policy:

- **Same-origin** requests always work
- **Cross-origin** requests require the server to send `Access-Control-Allow-Origin` headers
- You can set the request mode explicitly:

```rust
opts.mode(web_sys::RequestMode::Cors);  // default
opts.mode(web_sys::RequestMode::NoCors); // opaque response
```

The Wasm module runs with the **same origin restrictions** as the hosting page — no special privileges.

## Try It

Modify the function to include POST support with a JSON request body.
