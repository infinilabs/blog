---
title: "Shrinking a Search Engine to Fit in Your Browser — Part 1: Zero-Overhead WASM Bindings"
meta_title: "WASM Size Optimization Series Part 1"
description: "How we started cutting our WebAssembly search engine from 1.2 MB to under 300 KB by designing zero-overhead typed bindings and eliminating JSON from the WASM hot path."
date: "2026-05-10T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/wasm-series/part1-bindings.jpeg"
author: "Medcl"
tags: ["WebAssembly", "Rust", "Search Engine", "Performance", "Pizza Engine"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

Pizza Engine ships as a WebAssembly module that runs a full inverted-index search engine inside a browser tab or Node.js worker. You mount `.fire` segment files, run queries, get results — all without a network round-trip to a server.

This is Part 1 of a series documenting how we shrunk the WASM binary from **1.21 MB** (gzipped) to **245 KB** — an 80% reduction.

## The Problem: JSON is the Wrong Abstraction for WASM

The original API looked like this:

```js
const engine = new PizzaEngine(schemaJson);
engine.mount(segmentBytes);
const result = engine.search(
  '{"QueryString":{"query":"hello","default_field":"text"}}', 10
);
```

Simple on the surface. But behind that simplicity, **JSON is being parsed twice**:

1. JavaScript's `JSON.stringify` builds the query string
2. Rust's `serde_json::from_str` parses it back inside the WASM binary

And on the way back out:
1. Rust's `serde_wasm_bindgen::Serializer` walks the entire `SearchResult` tree
2. JavaScript receives a plain object and starts accessing fields

Both directions carry the full weight of a general-purpose JSON parser — one that inflates the WASM binary by ~200–300 KB compressed.

## The Insight: Let Each Side Do What It's Best At

JavaScript engines have **hardware-optimized JSON parsers**. V8's `JSON.parse` runs at memory bandwidth speeds. There's zero reason to ship a second JSON parser inside the WASM binary.

Meanwhile, `wasm-bindgen` already knows how to pass typed structs across the JS↔WASM boundary with zero copies. We just weren't using it.

The fix: **move JSON parsing to JavaScript, pass typed structs to Rust.**

## The New Architecture

### Before (JSON everywhere)

```bash
JS: JSON.stringify(query)  →  WASM: serde_json::from_str → Query → search
                                    SearchResult → serde_wasm_bindgen → JS Object
```

Every query pays: `serialize + deserialize + search + serialize + deserialize`

### After (typed boundary)

```bash
JS: new WasmQuery().queryString("hello", "text").size(10)
    → WASM: Query (already built) → search
    → WasmSearchResult: .total_hits (u32 getter), .hit(0).score (f32 getter)
```

Every query pays: `search`. That's it.

## The Typed API

### WasmQuery: Builder Pattern

```js
const q = new WasmQuery()
  .queryString("status:active AND category:electronics", "text")
  .size(20)
  .from(0)
  .aggTerms("top_brands", "brand", 10);
```

Each builder method constructs the Rust `Query` enum variant directly. No intermediate string. No allocation beyond the query struct itself.

### WasmSearchResult: Accessor Pattern

```js
const result = engine.searchTyped(q);
console.log(result.total_hits);  // u32 — zero-cost getter

for (let i = 0; i < result.hit_count; i++) {
  const hit = result.hit(i);
  console.log(hit.id, hit.score);      // u32, f32 — zero-cost
  console.log(hit.fieldStr("title"));  // single field extraction
}
```

No full-result serialization. Each hit's `_source` is materialized lazily — only if you ask for it. For paginated UIs displaying 10 results at a time, we serialize 10 documents instead of the entire result set.

### WasmSchemaBuilder: No JSON Schema Parsing

```js
const schema = new WasmSchemaBuilder()
  .addTextField("title")
  .addTextField("body")
  .addKeywordField("status")
  .addIntegerField("price")
  .build();

const engine = PizzaEngine.fromSchema(schema);
```

## Feature-Gated Tiers

We're not breaking anything. The implementation uses Rust's feature flags:

- **`wasm_nano`** — typed API only, no `serde_json` in binary
- **`wasm_mini`** — typed API + JSON API for DSL compatibility
- **`wasm_ultra`** — everything: remote loading, graph queries, explain

Existing code using the JSON API continues to work unchanged on mini/ultra tiers. The typed API is additive.

## Size Impact

The production WASM binary before any optimization:

| Metric | Size |
|--------|------|
| `.wasm` (release build) | 4.0 MB |
| Gzipped (network transfer) | **1.21 MB** |

Removable JSON machinery alone accounts for ~180 KB compressed.

## Protocol-Agnostic Core

The typed-boundary approach scales beyond WASM. The engine core (`Query`, `SearchResult`, `Schema`) is protocol-agnostic:

```bash
                    ┌──────────────┐
                    │ Engine Core  │
                    │  Query       │
                    │  Schema      │
                    │  SearchResult│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
    │wasm-bindgen│   │ Cap'n Proto│   │   JSON   │
    │  (typed)  │   │  (server) │   │  (REST)  │
    └───────────┘   └───────────┘   └──────────┘
```

## Progress

```bash
  1.21 MB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ original
  ~900 KB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━            ← Part 1
  <300 KB ━━━━━━━━━━━━━                                  ← goal
```

| ✅ | Milestone |
|---|---|
| ✅ | Identified 180 KB of removable JSON machinery |
| ✅ | Designed typed API: `WasmQuery`, `WasmSearchResult`, `WasmHit` |
| ✅ | Direct `js_sys::Object` construction (no serde) |
| ✅ | Split monolithic `wasm/mod.rs` into focused submodules |
| ✅ | Established 3-tier architecture: nano / mini / ultra |

**Result: 1.21 MB → ~900 KB gzipped (nano tier created)**

*Next: [Part 2 — Feature-Gated Aggregations](/posts/2026/05-11-wasm-series-2-feature-gated-aggregations/)*

---

*Pizza Engine is an embedded search engine written in Rust, targeting both server deployments (native binary) and edge/browser deployments (WebAssembly). The WASM bindings support tiered builds for different size/capability tradeoffs.*
