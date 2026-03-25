---
title: Building a Wasm Plugin System
slug: plugin-system
difficulty: advanced
tags: [getting-started]
order: 38
description: Design a plugin architecture using WebAssembly — sandboxed, portable, and language-agnostic — with trait-based host-guest communication and resource limits.
starter_code: |
  use std::collections::HashMap;

  // A plugin system in plain Rust demonstrating the patterns
  // used in real Wasm plugin architectures (Figma, VS Code, Envoy).

  // --- Plugin trait: the interface every plugin must implement ---
  trait Plugin {
      fn name(&self) -> &str;
      fn version(&self) -> &str;
      fn on_init(&mut self, config: &HashMap<String, String>);
      fn on_event(&mut self, event: &str, payload: &str) -> Option<String>;
      fn on_shutdown(&mut self);
  }

  // --- Host runtime: loads and manages plugins ---
  struct PluginHost {
      plugins: Vec<Box<dyn Plugin>>,
      event_log: Vec<String>,
      memory_limit: usize,   // bytes
      call_limit: u64,       // max calls per event
  }

  impl PluginHost {
      fn new(memory_limit: usize, call_limit: u64) -> Self {
          PluginHost {
              plugins: Vec::new(),
              event_log: Vec::new(),
              memory_limit,
              call_limit,
          }
      }

      fn register(&mut self, plugin: Box<dyn Plugin>) {
          println!("[host] Registered plugin: {} v{}", plugin.name(), plugin.version());
          self.plugins.push(plugin);
      }

      fn init_all(&mut self, config: &HashMap<String, String>) {
          println!("[host] Initializing {} plugin(s)...", self.plugins.len());
          for plugin in &mut self.plugins {
              plugin.on_init(config);
              println!("[host]   {} initialized", plugin.name());
          }
      }

      fn dispatch_event(&mut self, event: &str, payload: &str) {
          println!("[host] Dispatching event: '{}'", event);
          self.event_log.push(event.to_string());

          for plugin in &mut self.plugins {
              match plugin.on_event(event, payload) {
                  Some(response) => {
                      println!("[host]   {} responded: {}", plugin.name(), response);
                  }
                  None => {
                      println!("[host]   {} ignored event", plugin.name());
                  }
              }
          }
      }

      fn shutdown_all(&mut self) {
          println!("[host] Shutting down plugins...");
          for plugin in &mut self.plugins {
              plugin.on_shutdown();
          }
      }
  }

  // --- Example plugin: a word counter ---
  struct WordCountPlugin {
      total_words: usize,
  }

  impl WordCountPlugin {
      fn new() -> Self {
          WordCountPlugin { total_words: 0 }
      }
  }

  impl Plugin for WordCountPlugin {
      fn name(&self) -> &str { "word-counter" }
      fn version(&self) -> &str { "1.0.0" }

      fn on_init(&mut self, _config: &HashMap<String, String>) {
          self.total_words = 0;
      }

      fn on_event(&mut self, event: &str, payload: &str) -> Option<String> {
          if event == "text:input" {
              let count = payload.split_whitespace().count();
              self.total_words += count;
              Some(format!("words={} total={}", count, self.total_words))
          } else {
              None
          }
      }

      fn on_shutdown(&mut self) {
          println!("  [word-counter] Final count: {} words", self.total_words);
      }
  }

  // --- Example plugin: a keyword highlighter ---
  struct HighlightPlugin {
      keywords: Vec<String>,
  }

  impl HighlightPlugin {
      fn new(keywords: Vec<&str>) -> Self {
          HighlightPlugin {
              keywords: keywords.into_iter().map(|s| s.to_string()).collect(),
          }
      }
  }

  impl Plugin for HighlightPlugin {
      fn name(&self) -> &str { "highlighter" }
      fn version(&self) -> &str { "0.2.0" }

      fn on_init(&mut self, config: &HashMap<String, String>) {
          if let Some(extra) = config.get("highlight_extra") {
              self.keywords.push(extra.clone());
          }
      }

      fn on_event(&mut self, event: &str, payload: &str) -> Option<String> {
          if event == "text:input" {
              let found: Vec<&String> = self.keywords.iter()
                  .filter(|kw| payload.to_lowercase().contains(&kw.to_lowercase()))
                  .collect();
              if found.is_empty() {
                  None
              } else {
                  Some(format!("found keywords: {:?}", found))
              }
          } else {
              None
          }
      }

      fn on_shutdown(&mut self) {
          println!("  [highlighter] Shutting down");
      }
  }

  fn main() {
      println!("=== Wasm Plugin System Demo ===\n");

      // Create host with resource limits
      let mut host = PluginHost::new(1024 * 1024, 1000);

      // Register plugins
      host.register(Box::new(WordCountPlugin::new()));
      host.register(Box::new(HighlightPlugin::new(vec!["rust", "wasm"])));

      // Initialize with config
      let mut config = HashMap::new();
      config.insert("highlight_extra".to_string(), "plugin".to_string());
      host.init_all(&config);

      println!();

      // Dispatch events
      host.dispatch_event("text:input", "Rust and Wasm make great plugin hosts");
      println!();
      host.dispatch_event("text:input", "This has no keywords");
      println!();
      host.dispatch_event("file:save", "/path/to/file.txt");
      println!();

      // Shutdown
      host.shutdown_all();
  }
expected_output: |
  === Wasm Plugin System Demo ===

  [host] Registered plugin: word-counter v1.0.0
  [host] Registered plugin: highlighter v0.2.0
  [host] Initializing 2 plugin(s)...
  [host]   word-counter initialized
  [host]   highlighter initialized

  [host] Dispatching event: 'text:input'
  [host]   word-counter responded: words=7 total=7
  [host]   highlighter responded: found keywords: ["rust", "wasm", "plugin"]

  [host] Dispatching event: 'text:input'
  [host]   word-counter responded: words=4 total=11
  [host]   highlighter ignored event

  [host] Dispatching event: 'file:save'
  [host]   word-counter ignored event
  [host]   highlighter ignored event

  [host] Shutting down plugins...
    [word-counter] Final count: 11 words
    [highlighter] Shutting down
---

## Why Wasm for Plugins?

WebAssembly is becoming the standard runtime for plugin systems because it offers a unique combination of safety, performance, and portability:

```
  Traditional Plugin         Wasm Plugin
  (Native Code)              (Sandboxed)

  ┌─────────────────┐       ┌─────────────────────────────┐
  │  Host Process    │       │  Host Process               │
  │  ┌─────────────┐│       │  ┌────────────────────────┐ │
  │  │ Plugin (DLL) ││       │  │ Wasm Runtime           │ │
  │  │             ││       │  │ ┌──────────┐           │ │
  │  │ Full access ││       │  │ │ Plugin   │ sandboxed │ │
  │  │ to memory,  ││       │  │ │ .wasm    │ linear    │ │
  │  │ filesystem, ││       │  │ │          │ memory    │ │
  │  │ network ... ││       │  │ └──────────┘           │ │
  │  └─────────────┘│       │  │  No fs, no net, no     │ │
  │                  │       │  │  raw memory access     │ │
  │ ⚠ Crash = host  │       │  └────────────────────────┘ │
  │   crash          │       │  ✓ Crash = plugin only     │
  └─────────────────┘       └─────────────────────────────┘
```

| Property              | Native Plugin (DLL/SO) | Wasm Plugin             |
|-----------------------|------------------------|-------------------------|
| Sandboxed             | No                     | Yes                     |
| Language-agnostic     | No (ABI-dependent)     | Yes (any lang -> Wasm)  |
| Portable              | No (per-OS build)      | Yes (one binary)        |
| Near-native speed     | Yes                    | Yes (~0.8-0.95x native) |
| Memory safe           | Depends on language    | Yes (linear memory)     |
| Hot-reloadable        | Difficult              | Easy (re-instantiate)   |
| Resource limits       | Hard                   | Built-in (fuel, memory) |

## Real-World Examples

| Product      | How They Use Wasm Plugins                                  |
|-------------|-------------------------------------------------------------|
| **Figma**    | Runs community plugins in a Wasm sandbox inside the app    |
| **VS Code**  | Extension host can run Wasm extensions (web & desktop)     |
| **Envoy**    | Proxy filters as Wasm modules (network policy, auth, etc.) |
| **Shopify**  | Functions (discounts, shipping) run as Wasm               |
| **Zed**      | Editor extensions (themes, languages) as Wasm              |
| **Fermyon**  | Spin framework: entire microservices as Wasm components    |

## Architecture of a Wasm Plugin System

```
  ┌───────────────────────────────────────────────────────┐
  │                     Host Application                   │
  │                                                       │
  │  ┌─────────────┐  ┌────────────┐  ┌───────────────┐  │
  │  │ Plugin      │  │ Plugin     │  │ Plugin        │  │
  │  │ Registry    │  │ Loader     │  │ Lifecycle Mgr │  │
  │  │             │  │            │  │               │  │
  │  │ - manifest  │  │ - compile  │  │ - init()      │  │
  │  │ - versions  │  │ - validate │  │ - on_event()  │  │
  │  │ - deps      │  │ - instantiate│ │ - shutdown()  │  │
  │  └─────────────┘  └────────────┘  └───────────────┘  │
  │                         │                             │
  │                         ▼                             │
  │  ┌────────────────────────────────────────────────┐   │
  │  │              Wasm Runtime (wasmtime / wasmer)   │   │
  │  │                                                │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
  │  │  │Plugin A  │  │Plugin B  │  │Plugin C  │     │   │
  │  │  │ .wasm    │  │ .wasm    │  │ .wasm    │     │   │
  │  │  │          │  │          │  │          │     │   │
  │  │  │ 1MB mem  │  │ 2MB mem  │  │ 1MB mem  │     │   │
  │  │  │ limit    │  │ limit    │  │ limit    │     │   │
  │  │  └──────────┘  └──────────┘  └──────────┘     │   │
  │  └────────────────────────────────────────────────┘   │
  └───────────────────────────────────────────────────────┘
```

## Host-Guest Communication

The host and plugin communicate through **imported and exported functions**:

```
  Host                              Plugin (.wasm)
  ┌──────────────────┐             ┌──────────────────┐
  │                  │  exports    │                  │
  │  call plugin ────┼────────────>│ on_event()       │
  │  functions       │             │ on_init()        │
  │                  │             │ alloc() / free() │
  │                  │  imports    │                  │
  │  host_log()  <───┼────────────│ call host        │
  │  host_fetch()<───┼────────────│ functions         │
  │  host_kv_get()<──┼────────────│                  │
  └──────────────────┘             └──────────────────┘
```

### With wasmtime (Rust host)

```rust
use wasmtime::*;

fn load_plugin(engine: &Engine, path: &str) -> Result<Instance> {
    let module = Module::from_file(engine, path)?;
    let mut store = Store::new(engine, ());
    let mut linker = Linker::new(engine);

    // Provide host functions to the plugin
    linker.func_wrap("env", "host_log", |caller: Caller<'_, ()>, ptr: i32, len: i32| {
        // Read string from plugin's linear memory
        let memory = caller.get_export("memory").unwrap().into_memory().unwrap();
        let data = &memory.data(&caller)[ptr as usize..(ptr + len) as usize];
        let msg = std::str::from_utf8(data).unwrap();
        println!("[plugin log] {}", msg);
    })?;

    let instance = linker.instantiate(&mut store, &module)?;
    Ok(instance)
}
```

### Calling Plugin Functions

```rust
fn call_plugin_event(
    store: &mut Store<()>,
    instance: &Instance,
    event: &str,
) -> Result<i32> {
    let memory = instance.get_memory(&mut *store, "memory").unwrap();
    let alloc = instance.get_typed_func::<i32, i32>(&mut *store, "alloc")?;

    // Allocate space in plugin memory and copy the event string
    let ptr = alloc.call(&mut *store, event.len() as i32)?;
    memory.data_mut(&mut *store)[ptr as usize..ptr as usize + event.len()]
        .copy_from_slice(event.as_bytes());

    // Call the plugin's on_event function
    let on_event = instance
        .get_typed_func::<(i32, i32), i32>(&mut *store, "on_event")?;
    let result = on_event.call(&mut *store, (ptr, event.len() as i32))?;

    Ok(result)
}
```

## WIT: WebAssembly Interface Types

**WIT** (Wasm Interface Type) defines the contract between host and guest in a language-agnostic way:

```wit
// plugin.wit — the interface definition

package my-app:plugin@1.0.0;

interface host {
    /// Log a message to the host's console
    log: func(message: string);

    /// Read a key-value pair from host storage
    kv-get: func(key: string) -> option<string>;

    /// Write a key-value pair to host storage
    kv-set: func(key: string, value: string);
}

interface plugin {
    /// Called when the plugin is loaded
    init: func(config: list<tuple<string, string>>);

    /// Called for each event
    on-event: func(name: string, payload: string) -> option<string>;

    /// Called before the plugin is unloaded
    shutdown: func();
}

world my-plugin {
    import host;
    export plugin;
}
```

### wit-bindgen generates glue code:

```rust
// In the plugin crate (guest side)
wit_bindgen::generate!({
    world: "my-plugin",
    exports: {
        "my-app:plugin/plugin": MyPlugin,
    },
});

struct MyPlugin;

impl exports::my_app::plugin::plugin::Guest for MyPlugin {
    fn init(config: Vec<(String, String)>) {
        host::log(&format!("Initialized with {} config entries", config.len()));
    }

    fn on_event(name: String, payload: String) -> Option<String> {
        host::log(&format!("Got event: {}", name));
        Some(format!("processed: {}", payload))
    }

    fn shutdown() {
        host::log("Goodbye!");
    }
}
```

## Resource Limits and Sandboxing

Wasm runtimes provide fine-grained control over plugin resources:

### Memory Limits

```rust
let mut config = Config::new();
let engine = Engine::new(&config)?;

let memory_type = MemoryType::new(1, Some(16)); // 1 page min, 16 pages max (1 MB)
let memory = Memory::new(&mut store, memory_type)?;
```

### Fuel (CPU Limits)

```rust
let mut config = Config::new();
config.consume_fuel(true);
let engine = Engine::new(&config)?;

let mut store = Store::new(&engine, ());
store.set_fuel(1_000_000)?;  // 1M instructions

// Plugin runs until fuel is exhausted
let result = on_event.call(&mut store, (ptr, len));
match result {
    Err(e) if e.to_string().contains("fuel") => {
        println!("Plugin exceeded CPU budget!");
    }
    _ => {}
}
```

### Capability Table

```
  ┌──────────────────┬─────────┬─────────┬─────────┐
  │ Capability       │ Default │ Granted │ Denied  │
  ├──────────────────┼─────────┼─────────┼─────────┤
  │ Linear memory    │ 1 page  │ ≤ 16 pg │ > 16 pg │
  │ CPU (fuel)       │ 1M      │ Custom  │ 0       │
  │ Host functions   │ None    │ Linked  │ Omitted │
  │ File system      │ None    │ WASI    │ Default │
  │ Network          │ None    │ WASI    │ Default │
  │ System clock     │ None    │ WASI    │ Default │
  └──────────────────┴─────────┴─────────┴─────────┘
```

## Plugin Discovery and Loading

A practical system needs a way to discover, validate, and load plugins:

```rust
struct PluginManifest {
    name: String,
    version: String,
    wasm_path: String,
    permissions: Vec<String>,
    min_host_version: String,
}

struct PluginRegistry {
    manifests: HashMap<String, PluginManifest>,
    loaded: HashMap<String, Instance>,
}

impl PluginRegistry {
    fn discover(&mut self, plugin_dir: &str) -> Vec<String> {
        // Scan directory for plugin.toml files
        // Parse manifests, validate signatures
        // Return list of discovered plugin names
        vec![]
    }

    fn load(&mut self, name: &str, engine: &Engine) -> Result<(), String> {
        let manifest = self.manifests.get(name).ok_or("Not found")?;

        // Validate permissions
        for perm in &manifest.permissions {
            if !self.is_permission_allowed(perm) {
                return Err(format!("Permission denied: {}", perm));
            }
        }

        // Compile and instantiate
        // ...
        Ok(())
    }

    fn is_permission_allowed(&self, _perm: &str) -> bool { true }
}
```

### Hot Reloading

One of Wasm's strengths is easy hot-reload — drop the old instance, load the new one:

```rust
fn hot_reload(registry: &mut PluginRegistry, name: &str, engine: &Engine) {
    // 1. Call shutdown on old instance
    // 2. Drop old Instance (reclaims all memory)
    registry.loaded.remove(name);

    // 3. Load new .wasm file
    registry.load(name, engine).unwrap();

    // 4. Call init on new instance
    // Plugin state is fresh — no stale pointers or leaked memory
}
```

## Security Considerations

```
  Attack Surface Comparison:

  Native Plugin              Wasm Plugin
  ───────────────            ────────────────
  ✗ Buffer overflow          ✓ Linear memory bounds-checked
  ✗ Use-after-free           ✓ No raw pointers exposed
  ✗ Arbitrary syscalls       ✓ Only imported functions
  ✗ File system access       ✓ Explicit WASI capabilities
  ✗ Network access           ✓ Must be granted
  ✗ Shared memory corruption ✓ Isolated address space
```

| Threat                    | Mitigation                                  |
|---------------------------|---------------------------------------------|
| Infinite loop             | Fuel / instruction limits                   |
| Memory exhaustion         | Memory page limits                          |
| Malicious host calls      | Validate all arguments from plugin          |
| Data exfiltration         | No network unless explicitly granted        |
| Side-channel timing       | Disable clock access or add jitter          |

## Summary

Wasm plugin systems combine the performance of native code with the safety of sandboxed execution. The host defines a trait-like interface (or WIT contract), loads `.wasm` modules at runtime, and enforces resource limits. Real products like Figma, Envoy, and VS Code prove the model works at scale. Use `wasmtime` or `wasmer` as your runtime, `wit-bindgen` for ergonomic interfaces, and fuel/memory limits for safety.
