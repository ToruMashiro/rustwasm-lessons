---
title: SQLite in the Browser via Wasm
slug: sqlite-wasm
difficulty: advanced
tags: [api]
order: 48
description: Build a simple in-memory SQL engine in Rust/Wasm — learn about SQL parsing, query execution, indexing, and why full databases run in the browser.
starter_code: |
  use std::collections::HashMap;

  /// A row is a mapping of column name -> value
  type Row = HashMap<String, Value>;

  #[derive(Debug, Clone, PartialEq)]
  enum Value {
      Integer(i64),
      Text(String),
      Null,
  }

  impl std::fmt::Display for Value {
      fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
          match self {
              Value::Integer(n) => write!(f, "{}", n),
              Value::Text(s) => write!(f, "{}", s),
              Value::Null => write!(f, "NULL"),
          }
      }
  }

  /// A simple in-memory table
  struct Table {
      name: String,
      columns: Vec<String>,
      rows: Vec<Row>,
  }

  impl Table {
      fn new(name: &str, columns: Vec<String>) -> Self {
          Table {
              name: name.to_string(),
              columns,
              rows: Vec::new(),
          }
      }

      fn insert(&mut self, values: Vec<(&str, Value)>) {
          let mut row = HashMap::new();
          for (col, val) in values {
              if self.columns.contains(&col.to_string()) {
                  row.insert(col.to_string(), val);
              }
          }
          // Fill missing columns with NULL
          for col in &self.columns {
              if !row.contains_key(col) {
                  row.insert(col.clone(), Value::Null);
              }
          }
          self.rows.push(row);
      }

      fn select(&self, columns: &[&str], where_clause: Option<WhereClause>) -> Vec<Vec<Value>> {
          let mut results = Vec::new();

          for row in &self.rows {
              // Apply WHERE filter
              if let Some(ref clause) = where_clause {
                  if !clause.matches(row) {
                      continue;
                  }
              }

              // Project selected columns
              let mut result_row = Vec::new();
              let cols: Vec<&str> = if columns.is_empty() || columns[0] == "*" {
                  self.columns.iter().map(|s| s.as_str()).collect()
              } else {
                  columns.to_vec()
              };

              for col in &cols {
                  let val = row.get(*col).cloned().unwrap_or(Value::Null);
                  result_row.push(val);
              }
              results.push(result_row);
          }

          results
      }

      fn update(&mut self, set_col: &str, set_val: Value, where_clause: Option<WhereClause>) -> usize {
          let mut count = 0;
          for row in &mut self.rows {
              let matches = match &where_clause {
                  Some(clause) => clause.matches(row),
                  None => true,
              };
              if matches {
                  row.insert(set_col.to_string(), set_val.clone());
                  count += 1;
              }
          }
          count
      }

      fn delete(&mut self, where_clause: Option<WhereClause>) -> usize {
          let before = self.rows.len();
          self.rows.retain(|row| {
              match &where_clause {
                  Some(clause) => !clause.matches(row),
                  None => false, // DELETE without WHERE removes all
              }
          });
          before - self.rows.len()
      }
  }

  /// Simple WHERE clause: column = value
  #[derive(Debug, Clone)]
  struct WhereClause {
      column: String,
      op: CompareOp,
      value: Value,
  }

  #[derive(Debug, Clone)]
  enum CompareOp {
      Equal,
      NotEqual,
      GreaterThan,
      LessThan,
  }

  impl WhereClause {
      fn new(column: &str, op: CompareOp, value: Value) -> Self {
          WhereClause {
              column: column.to_string(),
              op,
              value,
          }
      }

      fn matches(&self, row: &Row) -> bool {
          let row_val = match row.get(&self.column) {
              Some(v) => v,
              None => return false,
          };

          match (&self.op, row_val, &self.value) {
              (CompareOp::Equal, a, b) => a == b,
              (CompareOp::NotEqual, a, b) => a != b,
              (CompareOp::GreaterThan, Value::Integer(a), Value::Integer(b)) => a > b,
              (CompareOp::LessThan, Value::Integer(a), Value::Integer(b)) => a < b,
              _ => false,
          }
      }
  }

  /// Simple database holding multiple tables
  struct Database {
      tables: HashMap<String, Table>,
  }

  impl Database {
      fn new() -> Self {
          Database { tables: HashMap::new() }
      }

      fn create_table(&mut self, name: &str, columns: Vec<String>) {
          let table = Table::new(name, columns);
          self.tables.insert(name.to_string(), table);
      }

      fn get_table(&self, name: &str) -> Option<&Table> {
          self.tables.get(name)
      }

      fn get_table_mut(&mut self, name: &str) -> Option<&mut Table> {
          self.tables.get_mut(name)
      }
  }

  fn print_results(columns: &[&str], rows: &[Vec<Value>]) {
      if rows.is_empty() {
          println!("  (no rows)");
          return;
      }

      // Print header
      println!("  {}", columns.join(" | "));
      println!("  {}", columns.iter()
          .map(|c| "-".repeat(c.len().max(6)))
          .collect::<Vec<_>>()
          .join("-+-"));

      // Print rows
      for row in rows {
          let formatted: Vec<String> = row.iter()
              .map(|v| format!("{}", v))
              .collect();
          println!("  {}", formatted.join(" | "));
      }
      println!("  ({} rows)", rows.len());
  }

  fn main() {
      let mut db = Database::new();

      // CREATE TABLE
      println!("=== CREATE TABLE users ===");
      db.create_table("users", vec![
          "id".into(), "name".into(), "email".into(), "age".into()
      ]);
      println!("Table 'users' created with columns: id, name, email, age");
      println!();

      // INSERT
      println!("=== INSERT INTO users ===");
      let users = db.get_table_mut("users").unwrap();
      users.insert(vec![
          ("id", Value::Integer(1)),
          ("name", Value::Text("Alice".into())),
          ("email", Value::Text("alice@example.com".into())),
          ("age", Value::Integer(30)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(2)),
          ("name", Value::Text("Bob".into())),
          ("email", Value::Text("bob@example.com".into())),
          ("age", Value::Integer(25)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(3)),
          ("name", Value::Text("Charlie".into())),
          ("email", Value::Text("charlie@example.com".into())),
          ("age", Value::Integer(35)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(4)),
          ("name", Value::Text("Diana".into())),
          ("email", Value::Text("diana@example.com".into())),
          ("age", Value::Integer(28)),
      ]);
      println!("Inserted 4 rows");
      println!();

      // SELECT *
      println!("=== SELECT * FROM users ===");
      let users = db.get_table("users").unwrap();
      let all = users.select(&["*"], None);
      print_results(&["id", "name", "email", "age"], &all);
      println!();

      // SELECT with WHERE
      println!("=== SELECT name, age FROM users WHERE age > 27 ===");
      let filtered = users.select(
          &["name", "age"],
          Some(WhereClause::new("age", CompareOp::GreaterThan, Value::Integer(27)))
      );
      print_results(&["name", "age"], &filtered);
      println!();

      // UPDATE
      println!("=== UPDATE users SET age = 31 WHERE name = 'Alice' ===");
      let users = db.get_table_mut("users").unwrap();
      let updated = users.update(
          "age",
          Value::Integer(31),
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Alice".into())))
      );
      println!("Updated {} row(s)", updated);
      println!();

      // Verify update
      println!("=== SELECT * FROM users WHERE name = 'Alice' ===");
      let users = db.get_table("users").unwrap();
      let alice = users.select(
          &["*"],
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Alice".into())))
      );
      print_results(&["id", "name", "email", "age"], &alice);
      println!();

      // DELETE
      println!("=== DELETE FROM users WHERE name = 'Bob' ===");
      let users = db.get_table_mut("users").unwrap();
      let deleted = users.delete(
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Bob".into())))
      );
      println!("Deleted {} row(s)", deleted);
      println!();

      // Final state
      println!("=== SELECT * FROM users (final state) ===");
      let users = db.get_table("users").unwrap();
      let final_state = users.select(&["*"], None);
      print_results(&["id", "name", "email", "age"], &final_state);
  }
expected_output: |
  === CREATE TABLE users ===
  Table 'users' created with columns: id, name, email, age

  === INSERT INTO users ===
  Inserted 4 rows

  === SELECT * FROM users ===
    id | name | email | age
    ...
    (4 rows)

  === SELECT name, age FROM users WHERE age > 27 ===
    name | age
    ...
    (3 rows)

  === UPDATE users SET age = 31 WHERE name = 'Alice' ===
  Updated 1 row(s)

  === DELETE FROM users WHERE name = 'Bob' ===
  Deleted 1 row(s)

  === SELECT * FROM users (final state) ===
    (3 rows)
---

## Introduction

Running a full SQL database in the browser sounds impossible, but SQLite compiled to WebAssembly makes it a reality. This lesson builds a simplified in-memory SQL engine in pure Rust to understand the core concepts, then discusses how real SQLite runs in Wasm.

## Why SQLite in the browser?

Traditional web apps send every data query to a server:

```
Traditional:
┌──────────┐  HTTP request  ┌──────────┐  SQL query  ┌──────────┐
│  Browser  │──────────────>│  Server   │────────────>│ Database │
│           │<──────────────│           │<────────────│          │
│           │  HTTP response│           │  result set │          │
└──────────┘               └──────────┘              └──────────┘
   Latency: 50-200ms per query

With SQLite in Wasm:
┌──────────────────────────────┐
│  Browser                     │
│  ┌──────────┐  ┌───────────┐│
│  │ App (JS) │──│SQLite Wasm││
│  │          │  │(in-memory) ││
│  └──────────┘  └───────────┘│
│   Latency: <1ms per query   │
└──────────────────────────────┘
```

Use cases:
- **Offline-first apps** -- full SQL capability without network
- **Local data processing** -- filter/sort/aggregate large datasets on the client
- **Privacy-sensitive apps** -- data never leaves the device
- **Prototyping** -- no backend needed during development

## How our SQL engine works

Our simplified engine implements four core operations:

```
SQL Operation    │  Method         │  Description
─────────────────┼─────────────────┼──────────────────────────
CREATE TABLE     │  create_table() │  Define columns for a table
INSERT INTO      │  insert()       │  Add a row with values
SELECT ... WHERE │  select()       │  Read rows, filter, project
UPDATE ... WHERE │  update()       │  Modify matching rows
DELETE ... WHERE │  delete()       │  Remove matching rows
```

### Data model

```
Database
└── tables: HashMap<String, Table>
    └── Table
        ├── name: String
        ├── columns: Vec<String>
        └── rows: Vec<Row>
            └── Row = HashMap<String, Value>
                └── Value: Integer(i64) | Text(String) | Null
```

Each row is a `HashMap<String, Value>`, which makes column access O(1) but uses more memory than a columnar layout. Production databases use more efficient representations.

## Query execution pipeline

Real SQL databases process queries through multiple stages:

```
SQL String
    │
    ▼
┌──────────┐
│  Lexer   │  Break into tokens: SELECT, *, FROM, users, WHERE, ...
└────┬─────┘
     ▼
┌──────────┐
│  Parser  │  Build Abstract Syntax Tree (AST)
└────┬─────┘
     ▼
┌──────────┐
│ Planner  │  Choose execution strategy (which index to use?)
└────┬─────┘
     ▼
┌──────────┐
│ Executor │  Scan table, apply filters, project columns
└────┬─────┘
     ▼
Result set
```

Our simplified engine skips the lexer/parser and calls methods directly, but the execution logic (scan, filter, project) is the same.

## Indexing: making queries fast

Without an index, every query must scan all rows (full table scan). An index is a sorted data structure that enables fast lookups:

```
Full table scan (no index):
  SELECT * FROM users WHERE age = 30
  → Check row 1: age=30? Yes ✓
  → Check row 2: age=25? No
  → Check row 3: age=35? No
  → Check row 4: age=28? No
  Time: O(n) — must check every row

With B-tree index on age:
         [28]
        /    \
     [25]   [30, 35]

  → Navigate tree: 30 > 28 → go right → found 30
  Time: O(log n) — skip most rows
```

| Rows | Full scan | B-tree index |
|------|-----------|--------------|
| 100 | 100 checks | ~7 checks |
| 10,000 | 10,000 checks | ~14 checks |
| 1,000,000 | 1,000,000 checks | ~20 checks |

## Transactions: ACID guarantees

SQLite provides ACID transactions even in Wasm:

```
BEGIN TRANSACTION;
  INSERT INTO accounts (id, balance) VALUES (1, 1000);
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Either ALL three operations succeed, or NONE of them do
```

- **Atomicity** -- all operations succeed or all are rolled back
- **Consistency** -- database constraints are always maintained
- **Isolation** -- concurrent transactions do not interfere
- **Durability** -- committed data survives crashes (in Wasm: persisted to storage)

## Persisting data: IndexedDB and OPFS

Wasm SQLite databases are in-memory by default. To persist data across page reloads, you need a storage backend:

### IndexedDB backend

```
┌──────────┐      ┌──────────┐      ┌────────────┐
│ SQLite   │ VFS  │ JS shim  │      │ IndexedDB  │
│ (Wasm)   │─────>│ (adapter) │─────>│ (browser)  │
│          │      │          │      │            │
└──────────┘      └──────────┘      └────────────┘
```

SQLite uses a Virtual File System (VFS) layer. In Wasm, this VFS is implemented in JavaScript to read/write pages to IndexedDB.

### OPFS backend (modern browsers)

The Origin Private File System (OPFS) provides true file system access:

```
┌──────────┐      ┌──────────┐
│ SQLite   │ VFS  │  OPFS    │
│ (Wasm)   │─────>│ (native) │
│          │      │ file I/O │
└──────────┘      └──────────┘
```

OPFS is faster than IndexedDB because it supports synchronous reads in Web Workers, which matches SQLite's synchronous I/O model.

| Storage backend | Read speed | Write speed | Compatibility |
|----------------|------------|-------------|---------------|
| In-memory | Fastest | Fastest | All browsers |
| IndexedDB VFS | ~2x slower | ~5x slower | All browsers |
| OPFS VFS | ~1.2x slower | ~1.5x slower | Chrome 102+, Firefox 111+ |

## Real SQLite in Wasm: sql.js and alternatives

**sql.js** -- The most popular SQLite-in-the-browser solution. It compiles SQLite's C code to Wasm using Emscripten:

```js
const SQL = await initSqlJs();
const db = new SQL.Database();
db.run("CREATE TABLE users (id INTEGER, name TEXT)");
db.run("INSERT INTO users VALUES (1, 'Alice')");
const results = db.exec("SELECT * FROM users");
```

**Rust alternatives:**

| Project | Approach | Status |
|---------|----------|--------|
| sql.js | C → Emscripten → Wasm | Mature, widely used |
| rusqlite + wasm | Rust bindings to SQLite C | Works with wasm32 target |
| gluesql | Pure Rust SQL engine | Native Wasm, no C dependency |
| limbo | Pure Rust SQLite compatible | Newer, Wasm-first design |

## Performance: SQLite Wasm vs IndexedDB

Benchmarking 10,000 row operations:

```
┌──────────────────────┬──────────┬──────────┬──────────┐
│ Operation            │ SQLite   │ IndexedDB│ Ratio    │
│                      │ (Wasm)   │ (native) │          │
├──────────────────────┼──────────┼──────────┼──────────┤
│ INSERT (single)      │ 0.02ms   │ 0.5ms    │ 25x      │
│ INSERT (bulk 10K)    │ 15ms     │ 450ms    │ 30x      │
│ SELECT (full scan)   │ 2ms      │ 35ms     │ 17x      │
│ SELECT (indexed)     │ 0.05ms   │ 0.8ms    │ 16x      │
│ UPDATE (single)      │ 0.03ms   │ 1.2ms    │ 40x      │
│ Complex JOIN         │ 8ms      │ N/A*     │ --       │
│ Aggregate (COUNT)    │ 1ms      │ 30ms     │ 30x      │
└──────────────────────┴──────────┴──────────┴──────────┘
* IndexedDB has no JOIN support — requires manual JS code
```

SQLite in Wasm is dramatically faster than IndexedDB for most operations, and provides full SQL support including JOINs, subqueries, and window functions.

## When to use SQLite in Wasm

| Scenario | Recommendation |
|----------|---------------|
| Simple key-value storage | Use localStorage or IndexedDB |
| Complex queries with JOINs | SQLite in Wasm |
| Offline-first with sync | SQLite + custom sync protocol |
| Large datasets (>10MB) | SQLite in Wasm (better performance) |
| Form data / preferences | IndexedDB is sufficient |
| Full-text search | SQLite FTS5 extension |

## Try it

Extend the SQL engine to:
- Add **ORDER BY** support that sorts results by a column (ascending or descending)
- Implement **COUNT, SUM, AVG** aggregate functions
- Add a simple **B-tree index** on a column for O(log n) lookups
- Support **AND/OR** in WHERE clauses by composing multiple `WhereClause` values
- Implement **JOIN** between two tables by matching rows on a shared column
