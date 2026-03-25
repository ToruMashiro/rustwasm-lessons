---
title: "Wasm + IndexedDB"
slug: wasm-indexeddb
difficulty: intermediate
tags: [api]
order: 32
description: Access IndexedDB from Rust via web-sys — create databases, perform CRUD operations, manage transactions, build indexes, and use JsFuture for async database calls.
starter_code: |
  use std::collections::HashMap;
  use std::fmt;

  fn main() {
      println!("=== Wasm + IndexedDB Simulation ===\n");

      // --- Simulating an IndexedDB-style key-value store ---

      #[derive(Debug, Clone)]
      struct Record {
          id: u32,
          name: String,
          email: String,
          age: u32,
      }

      impl fmt::Display for Record {
          fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
              write!(f, "{{ id: {}, name: {:?}, email: {:?}, age: {} }}",
                  self.id, self.name, self.email, self.age)
          }
      }

      struct ObjectStore {
          name: String,
          records: HashMap<u32, Record>,
          auto_increment: u32,
          indexes: HashMap<String, HashMap<String, Vec<u32>>>,
      }

      impl ObjectStore {
          fn new(name: &str) -> Self {
              println!("  Created object store: {:?}", name);
              ObjectStore {
                  name: name.to_string(),
                  records: HashMap::new(),
                  auto_increment: 1,
                  indexes: HashMap::new(),
              }
          }

          fn create_index(&mut self, index_name: &str) {
              self.indexes.insert(index_name.to_string(), HashMap::new());
              println!("  Created index: {:?} on store {:?}", index_name, self.name);
          }

          fn put(&mut self, mut record: Record) -> u32 {
              if record.id == 0 {
                  record.id = self.auto_increment;
                  self.auto_increment += 1;
              }
              let id = record.id;

              // Update indexes
              if self.indexes.contains_key("name") {
                  self.indexes.get_mut("name").unwrap()
                      .entry(record.name.clone())
                      .or_insert_with(Vec::new)
                      .push(id);
              }
              if self.indexes.contains_key("email") {
                  self.indexes.get_mut("email").unwrap()
                      .entry(record.email.clone())
                      .or_insert_with(Vec::new)
                      .push(id);
              }

              self.records.insert(id, record);
              id
          }

          fn get(&self, id: u32) -> Option<&Record> {
              self.records.get(&id)
          }

          fn delete(&mut self, id: u32) -> bool {
              self.records.remove(&id).is_some()
          }

          fn get_by_index(&self, index_name: &str, value: &str) -> Vec<&Record> {
              if let Some(index) = self.indexes.get(index_name) {
                  if let Some(ids) = index.get(value) {
                      return ids.iter()
                          .filter_map(|id| self.records.get(id))
                          .collect();
                  }
              }
              Vec::new()
          }

          fn count(&self) -> usize {
              self.records.len()
          }

          fn get_all(&self) -> Vec<&Record> {
              let mut records: Vec<&Record> = self.records.values().collect();
              records.sort_by_key(|r| r.id);
              records
          }
      }

      struct Database {
          name: String,
          version: u32,
          stores: HashMap<String, ObjectStore>,
      }

      impl Database {
          fn open(name: &str, version: u32) -> Self {
              println!("--- Opening Database ---");
              println!("  Database: {:?}, version: {}", name, version);
              Database {
                  name: name.to_string(),
                  version,
                  stores: HashMap::new(),
              }
          }

          fn create_object_store(&mut self, name: &str) -> &mut ObjectStore {
              self.stores.insert(name.to_string(), ObjectStore::new(name));
              self.stores.get_mut(name).unwrap()
          }

          fn transaction(&mut self, store_name: &str) -> Option<&mut ObjectStore> {
              self.stores.get_mut(store_name)
          }
      }

      // --- Open database and create schema ---
      let mut db = Database::open("my_app_db", 1);

      println!("\n--- Schema Setup (onupgradeneeded) ---");
      let store = db.create_object_store("users");
      store.create_index("name");
      store.create_index("email");

      // --- CREATE: Add records ---
      println!("\n--- CREATE Operations ---");
      let store = db.transaction("users").unwrap();

      let users = vec![
          Record { id: 0, name: "Alice".into(), email: "alice@example.com".into(), age: 30 },
          Record { id: 0, name: "Bob".into(), email: "bob@example.com".into(), age: 25 },
          Record { id: 0, name: "Charlie".into(), email: "charlie@example.com".into(), age: 35 },
          Record { id: 0, name: "Alice".into(), email: "alice2@example.com".into(), age: 28 },
      ];

      for user in users {
          let id = store.put(user.clone());
          println!("  PUT -> id={}: {} <{}>", id, user.name, user.email);
      }
      println!("  Store count: {}", store.count());

      // --- READ: Get by key ---
      println!("\n--- READ Operations ---");
      if let Some(record) = store.get(1) {
          println!("  GET(1): {}", record);
      }
      if let Some(record) = store.get(3) {
          println!("  GET(3): {}", record);
      }
      match store.get(99) {
          Some(r) => println!("  GET(99): {}", r),
          None => println!("  GET(99): not found"),
      }

      // --- READ: Get by index ---
      println!("\n--- INDEX Queries ---");
      let alices = store.get_by_index("name", "Alice");
      println!("  Index 'name'='Alice': {} results", alices.len());
      for r in &alices {
          println!("    {}", r);
      }

      let bob_emails = store.get_by_index("email", "bob@example.com");
      println!("  Index 'email'='bob@example.com': {} results", bob_emails.len());
      for r in &bob_emails {
          println!("    {}", r);
      }

      // --- UPDATE: Modify a record ---
      println!("\n--- UPDATE Operations ---");
      let updated = Record { id: 2, name: "Bob".into(), email: "bob.new@example.com".into(), age: 26 };
      store.put(updated);
      if let Some(record) = store.get(2) {
          println!("  Updated record 2: {}", record);
      }

      // --- DELETE: Remove a record ---
      println!("\n--- DELETE Operations ---");
      let deleted = store.delete(3);
      println!("  DELETE(3): success={}", deleted);
      println!("  Store count after delete: {}", store.count());

      let deleted_again = store.delete(99);
      println!("  DELETE(99): success={} (not found)", deleted_again);

      // --- List all records ---
      println!("\n--- All Remaining Records ---");
      for record in store.get_all() {
          println!("  {}", record);
      }

      // --- Transaction summary ---
      println!("\n--- Transaction Summary ---");
      println!("  Database: {:?} v{}", db.name, db.version);
      println!("  Object stores: {:?}", db.stores.keys().collect::<Vec<_>>());
      println!("  Total records: {}", db.transaction("users").unwrap().count());
  }
expected_output: |
  === Wasm + IndexedDB Simulation ===

  --- Opening Database ---
    Database: "my_app_db", version: 1

  --- Schema Setup (onupgradeneeded) ---
    Created object store: "users"
    Created index: "name" on store "users"
    Created index: "email" on store "users"

  --- CREATE Operations ---
    PUT -> id=1: Alice <alice@example.com>
    PUT -> id=2: Bob <bob@example.com>
    PUT -> id=3: Charlie <charlie@example.com>
    PUT -> id=4: Alice <alice2@example.com>
    Store count: 4

  --- READ Operations ---
    GET(1): { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
    GET(3): { id: 3, name: "Charlie", email: "charlie@example.com", age: 35 }
    GET(99): not found

  --- INDEX Queries ---
    Index 'name'='Alice': 2 results
      { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
      { id: 4, name: "Alice", email: "alice2@example.com", age: 28 }
    Index 'email'='bob@example.com': 1 results
      { id: 2, name: "Bob", email: "bob@example.com", age: 25 }

  --- UPDATE Operations ---
    Updated record 2: { id: 2, name: "Bob", email: "bob.new@example.com", age: 26 }

  --- DELETE Operations ---
    DELETE(3): success=true
    Store count after delete: 3
    DELETE(99): success=false (not found)

  --- All Remaining Records ---
    { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
    { id: 2, name: "Bob", email: "bob.new@example.com", age: 26 }
    { id: 4, name: "Alice", email: "alice2@example.com", age: 28 }

  --- Transaction Summary ---
    Database: "my_app_db" v1
    Object stores: ["users"]
    Total records: 3
---

## What Is IndexedDB?

IndexedDB is a low-level, transactional database built into every browser. Unlike localStorage (which only stores strings up to ~5 MB), IndexedDB can store structured data, binary blobs, and files with virtually no size limit.

```
┌─────────────────────────────────────────────────────┐
│                    Browser                           │
│                                                      │
│  ┌─────────────┐   ┌────────────────────────────┐   │
│  │  Wasm Module │──►│       IndexedDB             │   │
│  │  (Rust)      │   │                            │   │
│  │              │   │  Database: "my_app"         │   │
│  │  web-sys     │   │  ├── Store: "users"        │   │
│  │  bindings    │   │  │   ├── Index: "email"    │   │
│  │              │   │  │   └── Index: "name"     │   │
│  │  JsFuture    │   │  └── Store: "settings"     │   │
│  └─────────────┘   └────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

| Feature | localStorage | IndexedDB |
|---------|-------------|-----------|
| Data types | Strings only | Structured, binary, blobs |
| Size limit | ~5-10 MB | Hundreds of MB+ |
| Async | No (blocking) | Yes |
| Indexes | No | Yes |
| Transactions | No | Yes (ACID) |
| Cursors | No | Yes |

## Setting Up web-sys for IndexedDB

```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "IdbFactory",
    "IdbOpenDbRequest",
    "IdbDatabase",
    "IdbTransaction",
    "IdbTransactionMode",
    "IdbObjectStore",
    "IdbObjectStoreParameters",
    "IdbRequest",
    "IdbKeyRange",
    "IdbIndex",
    "IdbIndexParameters",
    "IdbCursorWithValue",
    "IdbCursorDirection",
    "DomException",
    "Event",
    "IdbVersionChangeEvent",
]
```

## Opening a Database

IndexedDB operations are callback-based in JavaScript. In Rust, we wrap them with `JsFuture` or manual `Promise` handling:

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{IdbDatabase, IdbOpenDbRequest, IdbObjectStoreParameters};
use wasm_bindgen_futures::JsFuture;

pub async fn open_db(name: &str, version: u32) -> Result<IdbDatabase, JsValue> {
    let window = web_sys::window().unwrap();
    let idb_factory = window.indexed_db()?.unwrap();

    let open_request: IdbOpenDbRequest = idb_factory.open_with_u32(name, version)?;

    // Handle schema upgrades
    let on_upgrade = Closure::wrap(Box::new(move |event: web_sys::IdbVersionChangeEvent| {
        let request: IdbOpenDbRequest = event.target().unwrap().dyn_into().unwrap();
        let db: IdbDatabase = request.result().unwrap().dyn_into().unwrap();

        // Create object stores during upgrade
        if !db.object_store_names().contains("users") {
            let mut params = IdbObjectStoreParameters::new();
            params.key_path(Some(&JsValue::from_str("id")));
            params.auto_increment(true);

            let store = db.create_object_store_with_optional_parameters("users", &params)?;
            store.create_index_with_str("email", "email")?;
            store.create_index_with_str("name", "name")?;
        }
    }) as Box<dyn FnMut(_)>);

    open_request.set_onupgradeneeded(Some(on_upgrade.as_ref().unchecked_ref()));
    on_upgrade.forget();

    // Wait for the database to open
    let result = JsFuture::from(open_request.into()).await?;
    Ok(result.dyn_into()?)
}
```

## CRUD Operations

### Create (put)

```rust
use serde::Serialize;

#[derive(Serialize)]
struct User {
    name: String,
    email: String,
    age: u32,
}

pub async fn add_user(db: &IdbDatabase, user: &User) -> Result<JsValue, JsValue> {
    let transaction = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    )?;

    let store = transaction.object_store("users")?;
    let js_value = serde_wasm_bindgen::to_value(user)?;
    let request = store.put(&js_value)?;

    JsFuture::from(request).await
}
```

### Read (get)

```rust
pub async fn get_user(db: &IdbDatabase, id: u32) -> Result<Option<JsValue>, JsValue> {
    let transaction = db.transaction_with_str("users")?;
    let store = transaction.object_store("users")?;
    let request = store.get(&JsValue::from(id))?;

    let result = JsFuture::from(request).await?;
    if result.is_undefined() {
        Ok(None)
    } else {
        Ok(Some(result))
    }
}
```

### Update

```rust
// put() with an existing key updates the record
pub async fn update_user(db: &IdbDatabase, user: &User) -> Result<JsValue, JsValue> {
    // Same as add_user — put() is an upsert operation
    add_user(db, user).await
}
```

### Delete

```rust
pub async fn delete_user(db: &IdbDatabase, id: u32) -> Result<(), JsValue> {
    let transaction = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    )?;

    let store = transaction.object_store("users")?;
    let request = store.delete(&JsValue::from(id))?;
    JsFuture::from(request).await?;
    Ok(())
}
```

## Transactions

IndexedDB transactions are **ACID-compliant**:

```
┌─────────────────────────────────────────┐
│            Transaction                   │
│                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │  put()   │  │  get()   │  │ delete() │ │
│  │ user #1  │  │ user #2  │  │ user #3  │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                          │
│  All succeed ──► COMMIT                  │
│  Any fails   ──► ROLLBACK (all undone)   │
└─────────────────────────────────────────┘
```

Transaction modes:

| Mode | Can Read? | Can Write? | Blocks Others? |
|------|----------|-----------|----------------|
| `readonly` | Yes | No | No |
| `readwrite` | Yes | Yes | Blocks writes to same stores |
| `versionchange` | Yes | Yes (schema) | Exclusive |

```rust
// Multiple operations in one transaction
pub async fn transfer_credits(
    db: &IdbDatabase,
    from_id: u32,
    to_id: u32,
    amount: u32,
) -> Result<(), JsValue> {
    let tx = db.transaction_with_str_and_mode(
        "users", web_sys::IdbTransactionMode::Readwrite,
    )?;
    let store = tx.object_store("users")?;

    // Both operations happen atomically
    let from = JsFuture::from(store.get(&from_id.into())?).await?;
    let to = JsFuture::from(store.get(&to_id.into())?).await?;

    // Modify and put back
    js_sys::Reflect::set(&from, &"credits".into(),
        &(get_credits(&from) - amount).into())?;
    js_sys::Reflect::set(&to, &"credits".into(),
        &(get_credits(&to) + amount).into())?;

    store.put(&from)?;
    store.put(&to)?;

    // Transaction auto-commits when all requests complete
    Ok(())
}
```

## Using Indexes for Queries

Indexes let you look up records by fields other than the primary key:

```rust
pub async fn find_by_email(
    db: &IdbDatabase,
    email: &str,
) -> Result<Option<JsValue>, JsValue> {
    let tx = db.transaction_with_str("users")?;
    let store = tx.object_store("users")?;
    let index = store.index("email")?;

    let request = index.get(&JsValue::from_str(email))?;
    let result = JsFuture::from(request).await?;

    if result.is_undefined() { Ok(None) } else { Ok(Some(result)) }
}
```

### Range Queries with IdbKeyRange

```rust
pub async fn find_users_in_age_range(
    db: &IdbDatabase,
    min_age: u32,
    max_age: u32,
) -> Result<Vec<JsValue>, JsValue> {
    let tx = db.transaction_with_str("users")?;
    let store = tx.object_store("users")?;
    let index = store.index("age")?;

    let range = web_sys::IdbKeyRange::bound(
        &JsValue::from(min_age),
        &JsValue::from(max_age),
    )?;

    let request = index.get_all_with_key(&range)?;
    let result = JsFuture::from(request).await?;
    let array: js_sys::Array = result.dyn_into()?;

    Ok(array.iter().collect())
}
```

## JsFuture: Bridging Callbacks to Async

IndexedDB was designed before Promises existed, so it uses event callbacks. `JsFuture` converts `IdbRequest` into a Rust `Future`:

```
┌────────────┐     ┌──────────────┐     ┌────────────┐
│ IdbRequest │────►│   Promise    │────►│  JsFuture   │
│ (callback) │     │  (wrapper)   │     │  (.await)   │
│            │     │              │     │             │
│ onsuccess  │     │  resolve()   │     │  Ok(value)  │
│ onerror    │     │  reject()    │     │  Err(error) │
└────────────┘     └──────────────┘     └────────────┘
```

```rust
use wasm_bindgen_futures::JsFuture;

// IdbRequest -> JsFuture (via implicit Promise conversion)
let request: IdbRequest = store.get(&key)?;
let result: JsValue = JsFuture::from(request).await?;
```

## Error Handling

```rust
pub async fn safe_db_operation(db: &IdbDatabase) -> Result<String, String> {
    let tx = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    ).map_err(|e| format!("Transaction failed: {:?}", e))?;

    let store = tx.object_store("users")
        .map_err(|e| format!("Store not found: {:?}", e))?;

    let request = store.get(&JsValue::from(1))
        .map_err(|e| format!("Get failed: {:?}", e))?;

    let result = JsFuture::from(request).await
        .map_err(|e| format!("Async error: {:?}", e))?;

    if result.is_undefined() {
        Err("Record not found".to_string())
    } else {
        Ok(format!("{:?}", result))
    }
}
```

## Key Takeaways

1. **IndexedDB** is a powerful browser database — transactional, indexed, handles binary data and large datasets
2. **web-sys** provides full IndexedDB bindings, but the API is verbose and callback-based
3. **JsFuture** bridges the callback-based API to Rust async/await
4. **Transactions** are ACID-compliant — group related operations for atomicity
5. **Indexes** enable efficient lookups on non-primary-key fields
6. **Schema upgrades** happen in the `onupgradeneeded` event — this is the only place you can create/delete stores and indexes
7. Always handle errors carefully — IndexedDB operations can fail due to quota limits, permissions, or schema mismatches
