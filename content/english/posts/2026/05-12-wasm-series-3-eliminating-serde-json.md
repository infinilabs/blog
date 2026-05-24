---
title: "Shrinking a Search Engine to Fit in Your Browser — Part 3: Eliminating serde_json"
meta_title: "WASM Size Optimization Series Part 3"
description: "Making serde_json an optional dependency and replacing it with a hand-rolled JSON emitter cuts another 18% — from 753 KB to 617 KB gzipped."
date: "2026-05-12T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/wasm-series/part3-serde.jpeg"
author: "Medcl"
tags: ["WebAssembly", "Rust", "Search Engine", "Performance", "Pizza Engine"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

This is Part 3 of a series on shrinking Pizza Engine's WASM binary from 1.21 MB to 245 KB. In [Part 2](/posts/2026/05-11-wasm-series-2-feature-gated-aggregations/), we gated aggregations. Now we remove the single heaviest dependency: `serde_json`.

## The Problem

After Part 2, nano clocked in at **753 KB gzipped**. Most of that weight comes from `serde_json` — a general-purpose JSON parser/serializer that the nano build doesn't actually need.

Why does nano ship `serde_json`? Because the query DSL input and search result output both route through JSON serialization. But nano targets a different use case: **pre-built segments with a known schema, simple keyword queries, typed results**. It doesn't need to parse arbitrary JSON DSL — it just needs to run text queries and return hits.

## The Approach

Make `serde_json` an optional dependency gated behind a `json` feature:

```toml
[dependencies]
serde_json = { version = "1.0", default-features = false, optional = true }

[features]
default = ["aggs", "json"]
json = ["dep:serde_json"]

wasm_nano = ["wasm", "wasm_panic_hook"]              # no json, no aggs
wasm_mini = ["wasm_nano", "json", "regex_queries", "wasm_dsl", "wasm_aggs"]
```

When `json` is disabled, `serde_json` is not compiled at all. Every call site that touches `serde_json::Value`, `serde_json::from_str`, or custom `Deserialize` impls is gated behind `#[cfg(feature = "json")]`.

## What Gets Gated

### WASM Entry Points

```rust
// Only available in mini/ultra:
#[cfg(feature = "json")]
#[wasm_bindgen]
pub fn search(&self, query_json: &str, k: usize) -> Result<JsValue, JsValue> { ... }

// Always available in nano:
#[wasm_bindgen]
pub fn search_text(&self, query: &str, field: &str, k: usize) -> Result<String, JsValue> { ... }
```

### Core Types — Dual Representation

Fields that previously held `serde_json::Value` get a no-JSON fallback:

```rust
#[cfg(feature = "json")]
pub origin: serde_json::Value,
#[cfg(not(feature = "json"))]
pub origin: String,
```

### Macro-Generated Deserialize

The query DSL macros generate custom `Deserialize` impls depending on `serde_json::Value`. Without `json`, they emit a simpler `#[derive(serde::Deserialize)]` instead.

## The Typed Nano API

Without JSON, nano builds results manually using only `alloc::format!`:

```rust
fn field_value_to_json(v: &FieldValue) -> String {
    match v {
        FieldValue::Text(s) => format!("\"{}\"", escape_json(s)),
        FieldValue::BigInt(n) => format!("{}", n),
        FieldValue::Double(n) => format!("{}", n),
        FieldValue::Boolean(b) => if *b { "true" } else { "false" }.to_string(),
        FieldValue::Array(arr) => /* recursive */,
        FieldValue::Object(map) => /* recursive */,
        FieldValue::Null => "null".to_string(),
        // ... all variants handled
    }
}
```

JavaScript uses its native `JSON.parse()` on the result — faster than any WASM-to-JS object marshaling.

### JavaScript Usage

```js
// Nano — typed API, no JSON parsing in WASM
const engine = new PizzaEngine();
engine.mount(bytes);
const json = engine.search_text("hello", "content", 10);
const result = JSON.parse(json);  // parsed on JS side — fast
```

## Design Decisions

**Why keep `serde` (without `_json`) as a hard dependency?**

`serde` itself is tiny — just traits and derive macros. The size cost is in `serde_json` (parser, formatter, `Value` type, number handling). Keeping `serde` means structs retain `#[derive(Serialize, Deserialize)]` for non-JSON serializers (bincode, postcard).

**Why return JSON strings from `search_text` instead of `JsValue`?**

Building `JsValue` requires `serde_wasm_bindgen` (heavier than `serde_json`) or manual `js_sys::Object` construction (many FFI calls). A pre-built JSON string lets JavaScript's native `JSON.parse()` do the work — faster for typical result sets (10–100 hits).

## Size Results

| Tier | Part 2 | Part 3 | Delta |
|------|--------|--------|-------|
| **nano** (gzip) | 753 KB | **617 KB** | **−18%** |
| **mini** (gzip) | 1.24 MB | 1.21 MB | same |

## Progress

```bash
  1.21 MB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ original
   753 KB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                 Part 2
   617 KB ━━━━━━━━━━━━━━━━━━━━━━━━━                      ← Part 3 (−49% total)
  <300 KB ━━━━━━━━━━━━━                                  ← goal
```

| ✅ | Milestone |
|---|---|
| ✅ | Made `serde_json` optional behind `json` feature |
| ✅ | Gated all JSON entry points (DSL parser, constructors, Document I/O) |
| ✅ | Dual-typed core structs (`serde_json::Value` → `String` fallback) |
| ✅ | Conditional macro codegen for Deserialize impls |
| ✅ | Typed `search_text` API with hand-rolled JSON emitter |
| ✅ | All tiers compile cleanly: nano, mini, ultra, native |

**Result: 753 KB → 617 KB gzipped (−18%, −49% cumulative)**

*Next: [Part 4 — Optional Geo & Vector Queries](/posts/2026/05-13-wasm-series-4-optional-geo-vector/)*
