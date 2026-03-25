---
title: Deploying to Production
slug: deploying
difficulty: advanced
tags: [getting-started]
order: 16
description: Bundle, optimize, and deploy Rust/Wasm applications to Cloudflare Pages, Vercel, Netlify, and other platforms.
starter_code: |
  fn main() {
      println!("=== Deployment Checklist ===\n");

      let checks = [
          ("Build mode", "wasm-pack build --release --target web"),
          ("Optimize binary", "wasm-opt -Oz -o opt.wasm pkg/app_bg.wasm"),
          ("Check size", "ls -lh pkg/*.wasm"),
          ("Test locally", "npx serve ."),
          ("Deploy", "npx wrangler pages deploy dist/"),
      ];

      for (i, (check, cmd)) in checks.iter().enumerate() {
          println!("  {}. {}", i + 1, check);
          println!("     $ {}\n", cmd);
      }

      println!("Size targets:");
      println!("  < 50KB  - Excellent");
      println!("  < 100KB - Good");
      println!("  < 500KB - Acceptable");
      println!("  > 1MB   - Needs optimization");
  }
expected_output: |
  === Deployment Checklist ===

    1. Build mode
       $ wasm-pack build --release --target web

    2. Optimize binary
       $ wasm-opt -Oz -o opt.wasm pkg/app_bg.wasm

    3. Check size
       $ ls -lh pkg/*.wasm

    4. Test locally
       $ npx serve .

    5. Deploy
       $ npx wrangler pages deploy dist/

  Size targets:
    < 50KB  - Excellent
    < 100KB - Good
    < 500KB - Acceptable
    > 1MB   - Needs optimization
---

## Build for Production

```bash
# Release build with size optimization
wasm-pack build --release --target web
```

Your `Cargo.toml` should have release optimizations:

```toml
[profile.release]
opt-level = "s"       # "s" = small, "z" = smallest
lto = true            # Link-time optimization
codegen-units = 1     # Better optimization
strip = true          # Remove debug symbols
panic = "abort"       # Smaller panic handling
```

## Optimize Binary Size

### Step 1: wasm-opt

```bash
# Install binaryen
npm install -g binaryen

# Optimize (can reduce size 10-30%)
wasm-opt -Oz -o optimized.wasm pkg/my_app_bg.wasm
```

### Step 2: Measure what's taking space

```bash
# Install twiggy
cargo install twiggy

# See top functions by size
twiggy top pkg/my_app_bg.wasm

# See call graph
twiggy paths pkg/my_app_bg.wasm
```

### Step 3: Remove unnecessary code

```rust
// Use #[cfg] to exclude code from Wasm builds
#[cfg(not(target_arch = "wasm32"))]
fn native_only() { /* ... */ }

// Avoid pulling in large crates for small features
// Use feature flags to minimize dependencies
```

## Serving Wasm Correctly

Your server must send the correct MIME type:

| File | Content-Type |
|------|-------------|
| `.wasm` | `application/wasm` |
| `.js` | `application/javascript` |

Most modern hosts handle this automatically. If not, add headers:

```
# _headers file (Cloudflare Pages, Netlify)
/*.wasm
  Content-Type: application/wasm
  Cache-Control: public, max-age=31536000, immutable
```

## Deploy to Cloudflare Pages

```bash
# Build
wasm-pack build --release --target web

# Create a dist/ folder with your HTML + pkg/
mkdir -p dist
cp index.html dist/
cp -r pkg dist/

# Deploy
npx wrangler pages deploy dist/ --project-name my-wasm-app
```

Or connect your GitHub repo for automatic deploys on every push.

## Deploy to Vercel

```json
// vercel.json
{
  "headers": [
    {
      "source": "/(.*).wasm",
      "headers": [
        { "key": "Content-Type", "value": "application/wasm" },
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

## Deploy to Netlify

```toml
# netlify.toml
[[headers]]
  for = "/*.wasm"
  [headers.values]
    Content-Type = "application/wasm"
    Cache-Control = "public, max-age=31536000, immutable"
```

## Caching Strategy

Wasm binaries are immutable once built — use aggressive caching:

```
# Cache .wasm for 1 year (content-hashed filename)
Cache-Control: public, max-age=31536000, immutable

# Don't cache the JS loader (might reference new .wasm)
Cache-Control: public, max-age=3600
```

## Loading Strategy

### Streaming compilation (fastest)

```js
// Browser compiles Wasm while downloading — fastest possible
const response = await fetch('app_bg.wasm');
const module = await WebAssembly.compileStreaming(response);
```

### Lazy loading

```js
// Only load Wasm when needed
let wasmModule = null;

async function getWasm() {
    if (!wasmModule) {
        wasmModule = await import('./pkg/my_app.js');
        await wasmModule.default();
    }
    return wasmModule;
}

// Load on first use
button.onclick = async () => {
    const wasm = await getWasm();
    wasm.process(data);
};
```

## Production Checklist

| Step | Command | Why |
|------|---------|-----|
| Release build | `wasm-pack build --release` | Remove debug info |
| wasm-opt | `wasm-opt -Oz` | Shrink binary 10-30% |
| Check size | `ls -lh pkg/*.wasm` | Target < 100KB |
| Test MIME type | Check Network tab | Must be `application/wasm` |
| Test HTTPS | Required for Wasm | Won't load over HTTP |
| Add caching | Set Cache-Control headers | 1 year for .wasm files |
| Error reporting | Add `console_error_panic_hook` | Debug production crashes |
| Fallback | Check `WebAssembly` exists | Show message if unsupported |

## Try It

Click **Run** to see the deployment checklist. Follow these steps when you're ready to ship your Rust/Wasm project.
