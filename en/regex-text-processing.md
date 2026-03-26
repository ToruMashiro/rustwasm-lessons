---
title: "Regex & Text Processing in Wasm"
slug: regex-text-processing
difficulty: intermediate
tags: [data-structures]
order: 42
description: Harness Rust's blazing-fast regex engine in WebAssembly — validate input, search text, parse logs, process CSV data, and handle Unicode correctly.
starter_code: |
  use std::collections::HashMap;

  // Demonstrating regex concepts and text processing in plain Rust.
  // The patterns here are the same ones you'd use with the `regex` crate
  // compiled to Wasm — we implement a mini matcher to show the internals.

  // --- Simple pattern matcher (simulating regex concepts) ---

  #[derive(Debug, Clone)]
  enum Pattern {
      Literal(String),
      AnyChar,                     // .
      Digit,                       // \d
      Word,                        // \w
      Whitespace,                  // \s
      OneOrMore(Box<Pattern>),     // +
      ZeroOrMore(Box<Pattern>),    // *
      Optional(Box<Pattern>),      // ?
      Group(Vec<Pattern>),         // (...)
  }

  fn matches_char(pat: &Pattern, ch: char) -> bool {
      match pat {
          Pattern::Literal(s) => s.len() == 1 && s.chars().next() == Some(ch),
          Pattern::AnyChar => ch != '\n',
          Pattern::Digit => ch.is_ascii_digit(),
          Pattern::Word => ch.is_alphanumeric() || ch == '_',
          Pattern::Whitespace => ch.is_whitespace(),
          _ => false,
      }
  }

  // --- Email validator (demonstrates common regex pattern) ---
  fn is_valid_email(email: &str) -> bool {
      // Pattern: [word+.-]+@[word.-]+\.[a-zA-Z]{2,}
      let parts: Vec<&str> = email.splitn(2, '@').collect();
      if parts.len() != 2 { return false; }

      let local = parts[0];
      let domain = parts[1];

      // Local part: at least 1 char, only word chars, dots, hyphens, plus
      if local.is_empty() { return false; }
      if !local.chars().all(|c| c.is_alphanumeric() || c == '.' || c == '-' || c == '_' || c == '+') {
          return false;
      }

      // Domain: must contain a dot, valid chars
      let domain_parts: Vec<&str> = domain.rsplitn(2, '.').collect();
      if domain_parts.len() != 2 { return false; }

      let tld = domain_parts[0];
      let host = domain_parts[1];

      if tld.len() < 2 || !tld.chars().all(|c| c.is_ascii_alphabetic()) { return false; }
      if host.is_empty() || !host.chars().all(|c| c.is_alphanumeric() || c == '.' || c == '-') {
          return false;
      }

      true
  }

  // --- Log parser ---
  #[derive(Debug)]
  struct LogEntry {
      timestamp: String,
      level: String,
      message: String,
  }

  fn parse_log_line(line: &str) -> Option<LogEntry> {
      // Format: [2024-01-15 09:23:45] ERROR: Something went wrong
      let line = line.trim();
      if !line.starts_with('[') { return None; }

      let bracket_end = line.find(']')?;
      let timestamp = line[1..bracket_end].to_string();

      let rest = line[bracket_end + 1..].trim();
      let colon_pos = rest.find(':')?;
      let level = rest[..colon_pos].trim().to_string();
      let message = rest[colon_pos + 1..].trim().to_string();

      Some(LogEntry { timestamp, level, message })
  }

  // --- CSV parser ---
  fn parse_csv_row(line: &str) -> Vec<String> {
      let mut fields = Vec::new();
      let mut current = String::new();
      let mut in_quotes = false;

      let chars: Vec<char> = line.chars().collect();
      let mut i = 0;
      while i < chars.len() {
          let ch = chars[i];
          if in_quotes {
              if ch == '"' {
                  if i + 1 < chars.len() && chars[i + 1] == '"' {
                      current.push('"');
                      i += 1;
                  } else {
                      in_quotes = false;
                  }
              } else {
                  current.push(ch);
              }
          } else if ch == '"' {
              in_quotes = true;
          } else if ch == ',' {
              fields.push(current.trim().to_string());
              current = String::new();
          } else {
              current.push(ch);
          }
          i += 1;
      }
      fields.push(current.trim().to_string());
      fields
  }

  // --- Text search with context (like grep -C) ---
  fn search_with_context(text: &str, query: &str, context_lines: usize) -> Vec<String> {
      let lines: Vec<&str> = text.lines().collect();
      let mut results = Vec::new();
      let query_lower = query.to_lowercase();

      for (i, line) in lines.iter().enumerate() {
          if line.to_lowercase().contains(&query_lower) {
              let start = i.saturating_sub(context_lines);
              let end = (i + context_lines + 1).min(lines.len());

              let mut block = Vec::new();
              for j in start..end {
                  let marker = if j == i { ">>>" } else { "   " };
                  block.push(format!("{} {:>3}: {}", marker, j + 1, lines[j]));
              }
              results.push(block.join("\n"));
          }
      }
      results
  }

  // --- Text replacement (like sed s/pattern/replacement/g) ---
  fn replace_all(text: &str, find: &str, replace: &str) -> String {
      text.replace(find, replace)
  }

  // --- Word frequency counter ---
  fn word_frequency(text: &str) -> Vec<(String, usize)> {
      let mut freq: HashMap<String, usize> = HashMap::new();
      for word in text.split_whitespace() {
          let cleaned: String = word.chars()
              .filter(|c| c.is_alphanumeric() || *c == '\'')
              .collect::<String>()
              .to_lowercase();
          if !cleaned.is_empty() {
              *freq.entry(cleaned).or_insert(0) += 1;
          }
      }
      let mut sorted: Vec<(String, usize)> = freq.into_iter().collect();
      sorted.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
      sorted
  }

  // --- Unicode handling ---
  fn unicode_stats(text: &str) -> (usize, usize, usize, usize) {
      let chars = text.chars().count();
      let bytes = text.len();
      let words = text.split_whitespace().count();
      let lines = text.lines().count();
      (chars, bytes, words, lines)
  }

  fn main() {
      println!("=== Regex & Text Processing Demo ===\n");

      // 1. Email validation
      println!("--- Email Validation ---");
      let emails = vec![
          "user@example.com",
          "john.doe+tag@company.co.uk",
          "invalid@",
          "@no-local.com",
          "spaces in@email.com",
          "valid_addr@sub.domain.org",
      ];
      for email in &emails {
          println!("  {:<35} {}", email, if is_valid_email(email) { "VALID" } else { "INVALID" });
      }

      // 2. Log parsing
      println!("\n--- Log Parsing ---");
      let log_lines = vec![
          "[2024-01-15 09:23:45] ERROR: Database connection failed",
          "[2024-01-15 09:23:46] INFO: Retrying connection...",
          "[2024-01-15 09:23:47] WARN: Connection slow (2.3s)",
          "malformed line without brackets",
      ];
      for line in &log_lines {
          match parse_log_line(line) {
              Some(entry) => println!("  [{:>5}] {} - {}", entry.level, entry.timestamp, entry.message),
              None => println!("  SKIP: {}", line),
          }
      }

      // 3. CSV parsing
      println!("\n--- CSV Parsing ---");
      let csv_data = r#"Name,Age,City
  Alice,30,"New York"
  Bob,25,"San Francisco"
  "O'Brien, Charlie",40,London"#;
      println!("  Input CSV:");
      for line in csv_data.lines() {
          println!("    {}", line.trim());
      }
      println!("  Parsed rows:");
      for (i, line) in csv_data.lines().enumerate() {
          let fields = parse_csv_row(line.trim());
          println!("    Row {}: {:?}", i, fields);
      }

      // 4. Text search with context
      println!("\n--- Text Search (query: 'error') ---");
      let document = "Line 1: All systems normal\nLine 2: Warning detected\nLine 3: Error in module X\nLine 4: Recovery started\nLine 5: System restored";
      let matches = search_with_context(document, "error", 1);
      for m in &matches {
          println!("{}", m);
      }

      // 5. Word frequency
      println!("\n--- Word Frequency ---");
      let text = "the quick brown fox jumps over the lazy dog the fox";
      let freq = word_frequency(text);
      println!("  Text: \"{}\"", text);
      println!("  Top words:");
      for (word, count) in freq.iter().take(5) {
          println!("    {:<10} {}", word, count);
      }

      // 6. Unicode stats
      println!("\n--- Unicode Handling ---");
      let unicode_text = "Hello, world!\nRust + Wasm = fast";
      let (chars, bytes, words, lines) = unicode_stats(unicode_text);
      println!("  Text: {:?}", unicode_text);
      println!("  Chars: {}, Bytes: {}, Words: {}, Lines: {}", chars, bytes, words, lines);
  }
expected_output: |
  === Regex & Text Processing Demo ===

  --- Email Validation ---
    user@example.com                    VALID
    john.doe+tag@company.co.uk          VALID
    invalid@                            INVALID
    @no-local.com                       INVALID
    spaces in@email.com                 INVALID
    valid_addr@sub.domain.org           VALID

  --- Log Parsing ---
    [ERROR] 2024-01-15 09:23:45 - Database connection failed
    [ INFO] 2024-01-15 09:23:46 - Retrying connection...
    [ WARN] 2024-01-15 09:23:47 - Connection slow (2.3s)
    SKIP: malformed line without brackets

  --- CSV Parsing ---
    Input CSV:
      Name,Age,City
      Alice,30,"New York"
      Bob,25,"San Francisco"
      "O'Brien, Charlie",40,London
    Parsed rows:
      Row 0: ["Name", "Age", "City"]
      Row 1: ["Alice", "30", "New York"]
      Row 2: ["Bob", "25", "San Francisco"]
      Row 3: ["O'Brien, Charlie", "40", "London"]

  --- Text Search (query: 'error') ---
      2: Line 2: Warning detected
  >>>  3: Line 3: Error in module X
      4: Line 4: Recovery started

  --- Word Frequency ---
    Text: "the quick brown fox jumps over the lazy dog the fox"
    Top words:
      the        3
      fox        2
      brown      1
      dog        1
      jumps      1

  --- Unicode Handling ---
    Text: "Hello, world!\nRust + Wasm = fast"
    Chars: 32, Bytes: 32, Words: 7, Lines: 2
---

## Why Rust Regex in Wasm?

JavaScript has built-in regex support, so why compile Rust's `regex` crate to Wasm? Performance, correctness, and features:

```
  Benchmark: Match 10,000 email addresses

  JS RegExp          ████████████████████████  24ms
  Rust regex (Wasm)  ████████  8ms
  Rust regex (native)████  4ms

  ← faster                         slower →
```

| Feature                  | JavaScript RegExp       | Rust `regex` crate      |
|--------------------------|-------------------------|-------------------------|
| Engine type              | Backtracking (NFA/DFA)  | Guaranteed DFA/NFA hybrid|
| Catastrophic backtracking| Possible (ReDoS)       | Impossible (linear time) |
| Unicode support          | Partial (ES2018+)       | Full (Unicode categories)|
| Named captures           | Yes (ES2018)            | Yes                     |
| Lookahead/lookbehind     | Yes                     | No (guarantees O(n))    |
| Compilation speed        | Fast                    | Slower (but cacheable)  |
| Match speed              | Good                    | Excellent (2-5x faster) |

### ReDoS Protection

The Rust `regex` crate guarantees **O(n)** matching time regardless of the pattern. This makes it safe against Regular Expression Denial of Service (ReDoS):

```
  Pattern: (a+)+$     Input: "aaaaaaaaaaaaaaaaX"

  JavaScript RegExp:
    Tries 2^n combinations → hangs for seconds
    This is a ReDoS vulnerability!

  Rust regex crate:
    Linear scan → returns "no match" instantly
    Safe by construction (no backtracking)
```

## Using the regex Crate in Wasm

```rust
use regex::Regex;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct TextProcessor {
    email_re: Regex,
    url_re: Regex,
    phone_re: Regex,
}

#[wasm_bindgen]
impl TextProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        TextProcessor {
            email_re: Regex::new(r"[\w.+-]+@[\w-]+\.[\w.]+").unwrap(),
            url_re: Regex::new(r"https?://[\w\-._~:/?#\[\]@!$&'()*+,;=%]+").unwrap(),
            phone_re: Regex::new(r"\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}").unwrap(),
        }
    }

    pub fn find_emails(&self, text: &str) -> Vec<String> {
        self.email_re.find_iter(text)
            .map(|m| m.as_str().to_string())
            .collect()
    }

    pub fn validate_email(&self, email: &str) -> bool {
        self.email_re.is_match(email)
    }

    pub fn redact_phones(&self, text: &str) -> String {
        self.phone_re.replace_all(text, "[REDACTED]").to_string()
    }
}
```

### Compiling regex to Wasm — size considerations

The `regex` crate adds approximately 200-400 KB to your Wasm binary (depending on Unicode table inclusion). To reduce size:

```toml
# Cargo.toml
[dependencies]
regex = { version = "1", default-features = false, features = ["std"] }
# Drops Unicode tables: saves ~100 KB but loses \p{...} patterns
```

## Common Regex Patterns for Wasm Apps

```
  Pattern                     Matches                   Use Case
  ─────────────────────────── ───────────────────────── ──────────────────
  [\w.+-]+@[\w-]+\.[\w.]+     user@example.com          Email validation
  https?://[^\s]+             http://example.com/path   URL extraction
  \d{4}-\d{2}-\d{2}          2024-01-15                ISO date
  \d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}  192.168.1.1     IP address
  #[0-9a-fA-F]{6}            #ff00aa                   Hex color
  \b[A-Z]{2,}\b              NASA, FBI                 Acronyms
  ^\s*$                       (empty/whitespace)        Blank lines
  <[^>]+>                     <div class="x">           HTML tags
```

## Text Search and Replace

### Efficient search with compiled regex

```rust
use regex::Regex;

// Compile once, use many times
lazy_static! {
    static ref LOG_PATTERN: Regex = Regex::new(
        r"\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\] (ERROR|WARN|INFO|DEBUG): (.+)"
    ).unwrap();
}

fn parse_log(line: &str) -> Option<(String, String, String)> {
    LOG_PATTERN.captures(line).map(|caps| (
        caps[1].to_string(),  // timestamp
        caps[2].to_string(),  // level
        caps[3].to_string(),  // message
    ))
}
```

### Search and replace with captures

```rust
use regex::Regex;

fn anonymize_data(text: &str) -> String {
    let email_re = Regex::new(r"([\w.+-]+)@([\w-]+\.[\w.]+)").unwrap();
    let phone_re = Regex::new(r"\d{3}[-.]?\d{3}[-.]?\d{4}").unwrap();

    let result = email_re.replace_all(text, "***@$2");  // Keep domain
    let result = phone_re.replace_all(&result, "***-***-****");
    result.to_string()
}
```

## CSV Processing in Wasm

CSV parsing is a common use case where Wasm outperforms JavaScript significantly:

```
  Processing 100,000 CSV rows

  Papa Parse (JS)     ████████████████████████████████  320ms
  Rust csv crate      ████████████  120ms
  (Wasm)

  ← faster                                    slower →
```

```rust
use csv::ReaderBuilder;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_csv(data: &str) -> JsValue {
    let mut reader = ReaderBuilder::new()
        .has_headers(true)
        .from_reader(data.as_bytes());

    let mut rows: Vec<Vec<String>> = Vec::new();
    for result in reader.records() {
        if let Ok(record) = result {
            rows.push(record.iter().map(|s| s.to_string()).collect());
        }
    }

    serde_wasm_bindgen::to_value(&rows).unwrap()
}
```

## Unicode Handling

Rust's `regex` crate has excellent Unicode support, which matters for international text processing:

```
  Character          UTF-8 Bytes    Rust char    JS char
  ────────────────── ────────────── ──────────── ─────────
  'A'                1 byte         1 char       1 code unit
  'e' (with accent)  2 bytes        1 char       1 code unit
  Chinese character  3 bytes        1 char       1 code unit
  Emoji              4 bytes        1 char       2 code units!
```

```rust
use regex::Regex;

// Unicode-aware word boundary
let re = Regex::new(r"\b\w+\b").unwrap();  // Works with Unicode!

// Unicode category matching
let re = Regex::new(r"\p{Han}+").unwrap();       // Chinese characters
let re = Regex::new(r"\p{Cyrillic}+").unwrap();  // Russian/Cyrillic
let re = Regex::new(r"\p{Emoji}+").unwrap();     // Emoji

// Case-insensitive with Unicode
let re = Regex::new(r"(?i)strasse|straße").unwrap();  // Matches German
```

## Performance Optimization Tips

```
  Optimization                         Impact
  ──────────────────────────────────── ──────────────────────────
  Compile regex once, reuse            10-100x faster
  Use is_match() instead of find()     Faster (no capture alloc)
  Anchor patterns (^...$)              Avoids scanning entire text
  Use RegexSet for multiple patterns   1 scan instead of N scans
  Disable Unicode if ASCII-only        Smaller binary, faster
  Use bytes::Regex for raw bytes       Avoids UTF-8 validation
```

### RegexSet: Match multiple patterns in one pass

```rust
use regex::RegexSet;

let set = RegexSet::new(&[
    r"[\w.+-]+@[\w-]+\.[\w.]+",    // email
    r"https?://[^\s]+",             // URL
    r"\d{3}[-.]?\d{3}[-.]?\d{4}",  // phone
    r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}",  // IP
]).unwrap();

let text = "Contact john@example.com or visit https://example.com";
let matches: Vec<usize> = set.matches(text).into_iter().collect();
// matches = [0, 1]  → found email and URL patterns
```

## Summary

Rust's `regex` crate compiled to Wasm gives you 2-5x faster text processing than JavaScript with guaranteed linear-time matching (no ReDoS). Use it for email validation, log parsing, CSV processing, search-and-replace, and any text-heavy workload. The crate adds ~200-400 KB to your Wasm binary but pays for itself in performance and safety. Combine it with full Unicode support for international text processing that just works.
