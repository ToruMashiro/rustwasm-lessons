---
title: WebSockets
slug: websockets
difficulty: intermediate
tags: [api]
order: 14
description: Build real-time communication with WebSockets from Rust/Wasm — connect, send, receive, and handle events.
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

          // Handle incoming messages
          let onmessage = Closure::wrap(Box::new(move |e: MessageEvent| {
              if let Some(text) = e.data().as_string() {
                  web_sys::console::log_1(
                      &format!("Received: {}", text).into()
                  );
              }
          }) as Box<dyn FnMut(MessageEvent)>);
          ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
          onmessage.forget();

          // Handle errors
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
  WebSocket connecting to ws://localhost:8080...
  Connected!
  Sent: "hello"
  Received: "echo: hello"
---

## Introduction

WebSockets enable real-time, bidirectional communication between the browser and a server. Rust/Wasm can manage the WebSocket connection, parse messages, and update application state — all with type safety.

## Setup

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

## Connecting

```rust
use web_sys::WebSocket;

let ws = WebSocket::new("wss://echo.websocket.org")?;

// Set binary mode for binary data
ws.set_binary_type(web_sys::BinaryType::Arraybuffer);
```

## Sending Messages

```rust
// Send text
ws.send_with_str("hello")?;

// Send binary data
let data: Vec<u8> = vec![1, 2, 3, 4];
let array = js_sys::Uint8Array::new_with_length(data.len() as u32);
array.copy_from(&data);
ws.send_with_array_buffer(&array.buffer())?;
```

## Receiving Messages

```rust
use web_sys::MessageEvent;
use wasm_bindgen::closure::Closure;

let onmessage = Closure::wrap(Box::new(move |e: MessageEvent| {
    // Text message
    if let Some(text) = e.data().as_string() {
        log!("Text: {}", text);
    }

    // Binary message
    if let Ok(buf) = e.data().dyn_into::<js_sys::ArrayBuffer>() {
        let array = js_sys::Uint8Array::new(&buf);
        let data = array.to_vec();
        log!("Binary: {} bytes", data.len());
    }
}) as Box<dyn FnMut(MessageEvent)>);

ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
onmessage.forget();
```

## Connection Lifecycle

```rust
// On open
let onopen = Closure::wrap(Box::new(move || {
    log!("Connected!");
}) as Box<dyn FnMut()>);
ws.set_onopen(Some(onopen.as_ref().unchecked_ref()));
onopen.forget();

// On close
let onclose = Closure::wrap(Box::new(move |e: web_sys::CloseEvent| {
    log!("Disconnected: code={}, reason={}", e.code(), e.reason());
}) as Box<dyn FnMut(_)>);
ws.set_onclose(Some(onclose.as_ref().unchecked_ref()));
onclose.forget();

// On error
let onerror = Closure::wrap(Box::new(move |e: ErrorEvent| {
    log!("Error: {}", e.message());
}) as Box<dyn FnMut(_)>);
ws.set_onerror(Some(onerror.as_ref().unchecked_ref()));
onerror.forget();
```

## Reconnection Pattern

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
            // Reconnect after delay
            let window = web_sys::window().unwrap();
            let url = url.clone();
            let closure = Closure::once(move || {
                log!("Reconnecting to {}...", url);
                // Re-create connection
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

## Use Cases

| Application | Why Wasm? |
|------------|----------|
| Chat app | Parse/validate messages in Rust, type-safe protocols |
| Live dashboard | Process streaming data (metrics, logs) at high throughput |
| Multiplayer game | Binary protocol parsing, state synchronization |
| Collaborative editor | OT/CRDT algorithms in Rust for correctness |

## Try It

The starter code shows a WebSocket client struct that connects, sends, receives, and handles errors. It wraps the browser WebSocket API with a clean Rust interface.
