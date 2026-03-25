---
title: "Closures & Callbacks Between JS and Rust"
slug: closures-callbacks
difficulty: intermediate
tags: [getting-started]
order: 29
description: Learn how Rust closures work across the Wasm boundary — Closure::wrap, Closure::once, the 'static lifetime requirement, passing functions between JS and Rust, and managing closure memory with forget() vs into_js_value().
starter_code: |
  fn main() {
      println!("=== Closures & Callbacks in Rust ===\n");

      // --- Basic closure syntax ---
      let greet = |name: &str| format!("Hello, {}!", name);
      println!("Basic closure: {}", greet("Wasm"));

      // --- Closures that capture variables ---
      let multiplier = 3;
      let multiply = |x: i32| x * multiplier;
      println!("Capture by ref: 5 * {} = {}", multiplier, multiply(5));

      // --- Move closures (ownership transfer) ---
      let prefix = String::from("LOG");
      let logger = move |msg: &str| format!("[{}] {}", prefix, msg);
      // prefix is now moved into the closure
      println!("Move closure: {}", logger("system started"));

      // --- FnOnce: can only be called once ---
      let data = vec![1, 2, 3, 4, 5];
      let consume = move || {
          let sum: i32 = data.iter().sum();
          println!("FnOnce consumed vec, sum = {}", sum);
          data  // returns ownership of data
      };
      let returned = consume();
      // consume() can't be called again — data was moved out
      println!("Returned data: {:?}", returned);

      // --- FnMut: can mutate captured state ---
      let mut counter = 0;
      let mut increment = || {
          counter += 1;
          counter
      };
      println!("\nFnMut counter:");
      for _ in 0..5 {
          print!("  count = {}", increment());
          println!();
      }

      // --- Fn: immutable borrow, can be called many times ---
      let threshold = 10;
      let check = |val: i32| -> bool { val > threshold };
      println!("\nFn check(5 > {}): {}", threshold, check(5));
      println!("Fn check(15 > {}): {}", threshold, check(15));

      // --- Simulating callback registration ---
      println!("\n--- Callback Simulation ---");

      fn register_callback<F: Fn(i32) -> i32>(name: &str, callback: F) {
          let result = callback(42);
          println!("Callback '{}' with input 42 => {}", name, result);
      }

      register_callback("double", |x| x * 2);
      register_callback("square", |x| x * x);
      register_callback("negate", |x| -x);

      // --- Boxed callbacks (heap-allocated, like Closure::wrap) ---
      println!("\n--- Boxed Callbacks (simulating Closure::wrap) ---");

      let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
          Box::new(|x| x + 1),
          Box::new(|x| x * 10),
          Box::new(|x| x - 5),
      ];

      for (i, cb) in callbacks.iter().enumerate() {
          println!("  boxed_callback[{}](100) = {}", i, cb(100));
      }

      // --- Simulating 'static lifetime requirement ---
      println!("\n--- 'static Lifetime Demo ---");

      fn needs_static<F: Fn() -> String + 'static>(f: F) -> String {
          f()
      }

      // This works: string literal is 'static
      let result = needs_static(|| String::from("I am 'static!"));
      println!("  'static closure result: {}", result);

      // This works: move captures owned data
      let owned = String::from("owned data");
      let result = needs_static(move || format!("Captured: {}", owned));
      println!("  move + 'static result: {}", result);

      // --- Simulating forget() vs drop behavior ---
      println!("\n--- Memory Management Simulation ---");

      struct ClosureWrapper {
          name: String,
      }

      impl ClosureWrapper {
          fn new(name: &str) -> Self {
              println!("  Created closure '{}'", name);
              ClosureWrapper { name: name.to_string() }
          }

          fn forget(self) {
              println!("  Forgot '{}' — leaked memory (never drops)", self.name);
              std::mem::forget(self);
          }
      }

      impl Drop for ClosureWrapper {
          fn drop(&mut self) {
              println!("  Dropped closure '{}'", self.name);
          }
      }

      let temp = ClosureWrapper::new("temporary");
      drop(temp);  // explicit drop

      let leaked = ClosureWrapper::new("event_listener");
      leaked.forget();  // simulates Closure::forget()

      let scoped = ClosureWrapper::new("scoped");
      // scoped drops automatically at end of scope
      println!("  (scoped will drop at end of block)");
  }
expected_output: |
  === Closures & Callbacks in Rust ===

  Basic closure: Hello, Wasm!
  Capture by ref: 5 * 3 = 15
  Move closure: [LOG] system started
  FnOnce consumed vec, sum = 15
  Returned data: [1, 2, 3, 4, 5]

  FnMut counter:
    count = 1
    count = 2
    count = 3
    count = 4
    count = 5

  Fn check(5 > 10): false
  Fn check(15 > 10): true

  --- Callback Simulation ---
  Callback 'double' with input 42 => 84
  Callback 'square' with input 42 => 1764
  Callback 'negate' with input 42 => -42

  --- Boxed Callbacks (simulating Closure::wrap) ---
    boxed_callback[0](100) = 101
    boxed_callback[1](100) = 1000
    boxed_callback[2](100) = 95

  --- 'static Lifetime Demo ---
    'static closure result: I am 'static!
    move + 'static result: Captured: owned data

  --- Memory Management Simulation ---
    Created closure 'temporary'
    Dropped closure 'temporary'
    Created closure 'event_listener'
    Forgot 'event_listener' — leaked memory (never drops)
    Created closure 'scoped'
    (scoped will drop at end of block)
    Dropped closure 'scoped'
---

## What Are Closures?

A **closure** is an anonymous function that can capture variables from its surrounding scope. In Rust, closures are one of the most powerful features — and they're essential for Wasm interop because JavaScript relies heavily on callbacks.

```rust
// Three equivalent ways to write a closure:
let add = |a, b| a + b;           // inferred types
let add = |a: i32, b: i32| a + b; // explicit types
let add = |a: i32, b: i32| -> i32 { a + b }; // full syntax
```

## The Three Closure Traits

Rust categorizes closures into three traits based on how they use captured variables:

```
+----------+-------------------+---------------------------+
|  Trait    |  Captures via     |  Can be called...         |
+----------+-------------------+---------------------------+
|  Fn      |  &T (immutable)   |  Many times               |
|  FnMut   |  &mut T (mutable) |  Many times (needs &mut)  |
|  FnOnce  |  T (by value)     |  Exactly once             |
+----------+-------------------+---------------------------+
```

The compiler automatically determines which trait a closure implements based on what it does with captured variables:

```rust
let x = 5;
let reads_x = || println!("{}", x);    // Fn (borrows x)
let mut y = 5;
let mutates_y = || { y += 1; };        // FnMut (mutably borrows y)
let z = String::from("hello");
let consumes_z = || { drop(z); };      // FnOnce (moves z)
```

**Important:** Every `FnOnce` is not necessarily `FnMut`, and every `FnMut` is not necessarily `Fn`. But `Fn` ⊂ `FnMut` ⊂ `FnOnce` (a closure implementing `Fn` also implements `FnMut` and `FnOnce`).

## The `move` Keyword

By default, closures capture variables by the smallest borrow needed. The `move` keyword forces the closure to take **ownership** of all captured variables:

```rust
let name = String::from("Alice");
let greet = move || println!("Hello, {}", name);
// name is no longer accessible here — it moved into the closure
greet();
```

This is critical for Wasm because closures passed to JavaScript must be `'static` — they can't borrow from the Rust stack because that stack frame may be gone when JS calls the closure.

## Closures in wasm-bindgen

In the Wasm world, `wasm_bindgen::closure::Closure` wraps a Rust closure so JavaScript can call it. There are two main constructors:

### Closure::wrap

```rust
// Closure::wrap creates a long-lived JS-callable closure
let cb = Closure::wrap(Box::new(|event: web_sys::MouseEvent| {
    // handle click
}) as Box<dyn FnMut(web_sys::MouseEvent)>);

element.add_event_listener_with_callback("click", cb.as_ref().unchecked_ref())?;
```

### Closure::once

```rust
// Closure::once creates a one-shot closure (FnOnce)
let cb = Closure::once(move || {
    // This runs once, then the closure is cleaned up
    web_sys::console::log_1(&"Loaded!".into());
});
```

## The `'static` Lifetime Requirement

When you pass a closure from Rust to JavaScript, it must satisfy the `'static` lifetime bound. This means:

```
┌─────────────────────────────────────────────────────┐
│                Rust Stack Frame                      │
│                                                      │
│  let local_string = String::from("hello");           │
│                                                      │
│  // WON'T COMPILE — closure borrows local_string     │
│  let bad = Closure::wrap(Box::new(|| {               │
│      console::log_1(&local_string.into());           │
│  }) as Box<dyn Fn()>);                               │
│                                                      │
│  // WORKS — move gives the closure ownership         │
│  let good = Closure::wrap(Box::new(move || {         │
│      console::log_1(&local_string.into());           │
│  }) as Box<dyn FnOnce()>);                           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

The reason: once the Rust function returns, its stack is gone. If JS later calls the closure, any borrowed references would be dangling. The `'static` bound ensures the closure owns everything it needs.

## Memory Management: forget() vs into_js_value()

This is one of the trickiest parts of Wasm closures. When a `Closure` is dropped in Rust, the JS callback becomes invalid. But you often need the callback to outlive the Rust function that created it.

### closure.forget()

```rust
let cb = Closure::wrap(Box::new(|| { /* ... */ }) as Box<dyn Fn()>);
element.set_onclick(Some(cb.as_ref().unchecked_ref()));
cb.forget();  // Leak memory — closure lives forever
```

**Pros:** Simple, always works.
**Cons:** Memory leak. If you add/remove event listeners frequently, memory grows unbounded.

### closure.into_js_value()

```rust
let cb = Closure::wrap(Box::new(|| { /* ... */ }) as Box<dyn Fn()>);
let js_func: JsValue = cb.into_js_value();
// js_func now owns the closure — it's freed when JS garbage-collects it
```

**Pros:** No memory leak — JS GC manages the lifetime.
**Cons:** You lose the Rust `Closure` handle (can't easily remove the listener later).

### Comparison Table

| Method | Memory Leak? | JS GC Manages? | Use Case |
|--------|-------------|----------------|----------|
| `forget()` | Yes | No | Long-lived event listeners |
| `into_js_value()` | No | Yes | Pass-and-forget callbacks |
| Store in struct | No | No | Controlled lifetime (best) |
| `Closure::once` | No | Auto-cleanup | One-shot callbacks |

## Best Practice: Store Closures in Structs

The cleanest approach is to store the `Closure` in a Rust struct that lives as long as the callback is needed:

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
            // handle click
        }) as Box<dyn FnMut(MouseEvent)>);

        element.set_onclick(Some(cb.as_ref().unchecked_ref()));
        self.click_handler = Some(cb);  // stored — not leaked
    }
}
// When App is dropped, click_handler is dropped, callback is invalidated
```

## Passing Functions FROM JavaScript to Rust

You can also accept JS functions as arguments:

```rust
#[wasm_bindgen]
pub fn call_js_function(f: &js_sys::Function) {
    let this = JsValue::NULL;
    let arg = JsValue::from(42);
    f.call1(&this, &arg).unwrap();
}
```

From JavaScript:
```js
import { call_js_function } from './pkg/my_module.js';
call_js_function((x) => console.log("Got from Rust:", x));
// Output: "Got from Rust: 42"
```

## Common Patterns

### Debounced Callback

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

### requestAnimationFrame Loop

```rust
fn request_animation_frame(f: &Closure<dyn FnMut()>) {
    window().unwrap()
        .request_animation_frame(f.as_ref().unchecked_ref())
        .expect("should register `requestAnimationFrame`");
}

let f = Rc::new(RefCell::new(None));
let g = f.clone();

*g.borrow_mut() = Some(Closure::wrap(Box::new(move || {
    // render frame here
    request_animation_frame(f.borrow().as_ref().unwrap());
}) as Box<dyn FnMut()>));

request_animation_frame(g.borrow().as_ref().unwrap());
```

## Key Takeaways

1. **Rust closures** implement `Fn`, `FnMut`, or `FnOnce` depending on how they capture variables
2. **`move` closures** are almost always needed for Wasm because of the `'static` requirement
3. **`Closure::wrap`** creates multi-use JS callbacks; **`Closure::once`** creates single-use ones
4. **`forget()`** leaks memory but is simple; **`into_js_value()`** lets JS GC manage lifetime
5. **Best practice:** store `Closure` values in a struct field so their lifetime is explicitly managed
6. **JS → Rust:** accept `&js_sys::Function` to receive JavaScript functions in Rust
