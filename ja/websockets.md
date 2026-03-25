---
title: WebSocket
slug: websockets
difficulty: intermediate
tags: [api]
order: 14
description: Rust/WasmからWebSocketを使ったリアルタイム通信を構築します — 接続、送信、受信、イベントハンドリング。
starter_code: |
  use wasm_bindgen::prelude::*;
  use web_sys::{WebSocket, MessageEvent, ErrorEvent};
  use wasm_bindgen::closure::Closure;

  #[wasm_bindgen]
  pub struct WsClient {
      ws: WebSocket,
  }

  #[wasm_bindgen]
  impl WsClient {
      #[wasm_bindgen(constructor)]
      pub fn new(url: &str) -> Result<WsClient, JsValue> {
          let ws = WebSocket::new(url)?;

          // 受信メッセージのハンドリング
          let onmessage = Closure::wrap(Box::new(move |e: MessageEvent| {
              if let Some(text) = e.data().as_string() {
                  web_sys::console::log_1(
                      &format!("Received: {}", text).into()
                  );
              }
          }) as Box<dyn FnMut(MessageEvent)>);
          ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
          onmessage.forget();

          // エラーのハンドリング
          let onerror = Closure::wrap(Box::new(move |e: ErrorEvent| {
              web_sys::console::error_1(&e.message().into());
          }) as Box<dyn FnMut(ErrorEvent)>);
          ws.set_onerror(Some(onerror.as_ref().unchecked_ref()));
          onerror.forget();

          Ok(WsClient { ws })
      }

      pub fn send(&self, msg: &str) -> Result<(), JsValue> {
          self.ws.send_with_str(msg)
      }

      pub fn close(&self) -> Result<(), JsValue> {
          self.ws.close()
      }
  }
expected_output: |
  WebSocket接続中 ws://localhost:8080...
  接続完了！
  送信: "hello"
  受信: "echo: hello"
---

## はじめに

WebSocketは、ブラウザとサーバー間のリアルタイムな双方向通信を実現します。Rust/WasmでWebSocket接続の管理、メッセージのパース、アプリケーション状態の更新を行えます — すべて型安全に。

## セットアップ

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "WebSocket",
    "MessageEvent",
    "ErrorEvent",
    "CloseEvent",
    "BinaryType",
    "console",
]
```

## 接続

```rust
use web_sys::WebSocket;

let ws = WebSocket::new("wss://echo.websocket.org")?;

// バイナリデータ用のバイナリモードを設定
ws.set_binary_type(web_sys::BinaryType::Arraybuffer);
```

## メッセージの送信

```rust
// テキストの送信
ws.send_with_str("hello")?;

// バイナリデータの送信
let data: Vec<u8> = vec![1, 2, 3, 4];
let array = js_sys::Uint8Array::new_with_length(data.len() as u32);
array.copy_from(&data);
ws.send_with_array_buffer(&array.buffer())?;
```

## メッセージの受信

```rust
use web_sys::MessageEvent;
use wasm_bindgen::closure::Closure;

let onmessage = Closure::wrap(Box::new(move |e: MessageEvent| {
    // テキストメッセージ
    if let Some(text) = e.data().as_string() {
        log!("Text: {}", text);
    }

    // バイナリメッセージ
    if let Ok(buf) = e.data().dyn_into::<js_sys::ArrayBuffer>() {
        let array = js_sys::Uint8Array::new(&buf);
        let data = array.to_vec();
        log!("Binary: {} bytes", data.len());
    }
}) as Box<dyn FnMut(MessageEvent)>);

ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
onmessage.forget();
```

## 接続のライフサイクル

```rust
// 接続時
let onopen = Closure::wrap(Box::new(move || {
    log!("Connected!");
}) as Box<dyn FnMut()>);
ws.set_onopen(Some(onopen.as_ref().unchecked_ref()));
onopen.forget();

// 切断時
let onclose = Closure::wrap(Box::new(move |e: web_sys::CloseEvent| {
    log!("Disconnected: code={}, reason={}", e.code(), e.reason());
}) as Box<dyn FnMut(_)>);
ws.set_onclose(Some(onclose.as_ref().unchecked_ref()));
onclose.forget();

// エラー時
let onerror = Closure::wrap(Box::new(move |e: ErrorEvent| {
    log!("Error: {}", e.message());
}) as Box<dyn FnMut(_)>);
ws.set_onerror(Some(onerror.as_ref().unchecked_ref()));
onerror.forget();
```

## 再接続パターン

```rust
#[wasm_bindgen]
pub struct ReconnectingWs {
    url: String,
    ws: Option<WebSocket>,
    retry_count: u32,
}

#[wasm_bindgen]
impl ReconnectingWs {
    pub fn connect(&mut self) -> Result<(), JsValue> {
        let ws = WebSocket::new(&self.url)?;

        let url = self.url.clone();
        let onclose = Closure::wrap(Box::new(move || {
            // 遅延後に再接続
            let window = web_sys::window().unwrap();
            let url = url.clone();
            let closure = Closure::once(move || {
                log!("Reconnecting to {}...", url);
                // 接続を再作成
            });
            window.set_timeout_with_callback_and_timeout_and_arguments_0(
                closure.as_ref().unchecked_ref(), 3000
            ).unwrap();
            closure.forget();
        }) as Box<dyn FnMut()>);

        ws.set_onclose(Some(onclose.as_ref().unchecked_ref()));
        onclose.forget();

        self.ws = Some(ws);
        Ok(())
    }
}
```

## ユースケース

| アプリケーション | Wasmを使う理由 |
|-----------------|---------------|
| チャットアプリ | Rustでメッセージをパース・検証、型安全なプロトコル |
| ライブダッシュボード | ストリーミングデータ（メトリクス、ログ）を高スループットで処理 |
| マルチプレイヤーゲーム | バイナリプロトコルのパース、状態の同期 |
| 共同編集エディタ | 正確性のためにRustでOT/CRDTアルゴリズムを実装 |

## 試してみよう

スターターコードは、接続、送信、受信、エラーハンドリングを行うWebSocketクライアント構造体を示しています。ブラウザのWebSocket APIをクリーンなRustインターフェースでラップしています。
