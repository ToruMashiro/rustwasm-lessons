---
title: DOM Manipulation
slug: dom-manipulation
difficulty: beginner
tags: [dom, getting-started]
order: 5
description: Access and modify the browser DOM directly from Rust using web-sys bindings.
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
  Created <div> element with text "Hello from Rust!"
  Element added to document body
---

## Introduction

The `web-sys` crate provides Rust bindings for **all** Web APIs — DOM, Canvas, WebGL, Fetch, WebSockets, and more. These bindings are auto-generated from the WebIDL specifications, so they match the browser APIs exactly.

## How web-sys works

`web-sys` is a thin wrapper around JavaScript Web APIs. Each API must be enabled as a Cargo feature:

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

Only the features you list are compiled — keeping the binary small.

## Accessing the DOM

The entry points mirror JavaScript exactly:

```rust
use web_sys::window;

// window → document → body (same chain as JS)
let window = window().expect("no global window");
let document = window.document().expect("no document");
let body = document.body().expect("no body");
```

Note: `window()`, `document()`, and `body()` return `Option` — they can fail if called in a non-browser environment (e.g., Node.js without jsdom).

## Creating and modifying elements

```rust
// Create a new element
let div = document.create_element("div")?;

// Set content and attributes
div.set_text_content(Some("Hello from Rust!"));
div.set_attribute("id", "my-div")?;
div.set_attribute("class", "highlight")?;

// Set inner HTML (use carefully — XSS risk with user input)
div.set_inner_html("<strong>Bold text</strong>");

// Append to DOM
body.append_child(&div)?;
```

## Querying elements

```rust
// By ID
let el = document.get_element_by_id("my-div");

// By CSS selector (returns first match)
let el = document.query_selector(".highlight")?;

// All matches
let list = document.query_selector_all("div.item")?;
for i in 0..list.length() {
    let node = list.get(i).unwrap();
    // ... process each node
}
```

## Adding Event Listeners

Use `Closure::wrap` to turn a Rust closure into a JavaScript callback:

```rust
use wasm_bindgen::closure::Closure;
use web_sys::HtmlElement;

let button: HtmlElement = document
    .get_element_by_id("my-btn")
    .unwrap()
    .dyn_into()?;

let closure = Closure::wrap(Box::new(move |_event: web_sys::MouseEvent| {
    web_sys::console::log_1(&"Clicked!".into());
}) as Box<dyn FnMut(_)>);

button.add_event_listener_with_callback(
    "click",
    closure.as_ref().unchecked_ref(),
)?;

// IMPORTANT: prevent Rust from dropping the closure
// (otherwise the callback becomes invalid)
closure.forget();
```

**Why `closure.forget()`?** Rust normally drops values when they go out of scope. But this closure is now owned by the browser's event system. Calling `.forget()` tells Rust not to deallocate it. For long-lived callbacks this is fine — for short-lived ones, store the `Closure` in a struct to control its lifetime.

## Common web-sys features

| Feature | Provides | Use case |
|---------|---------|----------|
| `"Window"` | `window()`, timers, location | Entry point |
| `"Document"` | DOM queries, element creation | DOM manipulation |
| `"Element"` | Attributes, classes, children | Element modification |
| `"HtmlElement"` | Style, click, focus | Interactive elements |
| `"HtmlCanvasElement"` | Canvas 2D/WebGL | Graphics |
| `"console"` | `console::log_1()` | Debugging |
| `"Storage"` | `local_storage()` | Persistence |
| `"MouseEvent"` | Click coordinates, buttons | Event handling |
| `"KeyboardEvent"` | Key codes, modifiers | Keyboard input |
| `"RequestInit"` | Fetch options | HTTP requests |

## Error handling pattern

Most web-sys methods return `Result<T, JsValue>`. Use `?` to propagate errors:

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

## Try It

Try creating a `<button>` element and adding a click event listener using `Closure::wrap`.
