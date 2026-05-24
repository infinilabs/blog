---
title: "Shrinking a Search Engine to Fit in Your Browser — Part 4: Optional Geo & Vector Queries"
meta_title: "WASM Size Optimization Series Part 4"
description: "Feature-gating geo-spatial and vector search dispatch brings the nano binary to 245 KB gzipped — achieving our sub-300 KB goal with an 80% total reduction."
date: "2026-05-13T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/wasm-series/part4-geo-vector.jpeg"
author: "Medcl"
tags: ["WebAssembly", "Rust", "Search Engine", "Performance", "Pizza Engine"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

This is Part 4 (final) of a series on shrinking Pizza Engine's WASM binary from 1.21 MB to 245 KB. In [Part 3](/posts/2026/05-12-wasm-series-3-eliminating-serde-json/), we eliminated `serde_json`. Now we gate geo-spatial and vector queries as optional features, lock in the regex gating from Part 3's follow-up work, and celebrate hitting our target.

## The Problem

Pizza Engine supports geo-spatial queries (bounding box, distance) backed by a BKD tree (~4000 LOC), and vector/KNN queries backed by brute-force flat scan (~500 LOC). Both are essential for production — but a nano deployment that only does keyword lookups can never invoke them, since those query types require JSON DSL input unavailable in the nano tier.

## The Approach

Two independent feature flags control whether geo and vector execution paths are compiled:

```toml
[features]
geo_queries = []     # Geo bounding-box and geo-distance query support
vector_queries = []  # Dense/sparse vector (KNN) query support
regex_queries = ["dep:fst-regex"]  # Regexp query support

wasm_nano = ["wasm", "wasm_panic_hook"]  # none of the above
wasm_mini = ["wasm_nano", "json", "regex_queries", "geo_queries",
             "vector_queries", "query_string_parser", "wasm_dsl", "wasm_aggs"]
```

Nano excludes all three. Mini includes all three. Users can compose custom tiers.

## Why Not Gate the Modules Themselves?

The BKD tree module is imported in 30+ files across the codebase. Adding `#[cfg(feature = "geo_queries")]` to the module declaration produces **46 cascading compile errors** — every `use` statement and match arm referencing the module breaks.

This is a common challenge with deeply-integrated code in monolithic crates. The cost of full module gating is disproportionate to the benefit.

## What Works: LTO + Entry-Point Guards

### Structural Unreachability

Nano's only public API is `search_text(query_string, field, k)`, which calls `parse_query_string_to_query`. This parser produces basic text queries:

- `Query::Term`, `Query::Match`, `Query::Prefix`, `Query::Wildcard`, `Query::Bool`

It **never** produces:

- `Query::Vector(...)` — requires structured JSON
- `Query::GeoBoundingBox(...)` — requires structured JSON
- `Query::GeoDistance(...)` — requires structured JSON

Since no code path in nano constructs these variants, LTO (`lto = true`, `codegen-units = 1`) traces reachability and eliminates all downstream implementation code.

### Explicit `#[cfg]` at the Execution Boundary

For defense-in-depth, we gate the lowest-level dispatch point — where a query crosses from pattern-matching into real computation:

```rust
// src/index/immutable/inverted/mmap/leaf_producer.rs

#[cfg(feature = "geo_queries")]
if let Some(ids) = self.segment.range_index_query_doc_ids(self.schema, query) {
    if ids.is_empty() { return Ok(None); }
    let hits = ids.into_iter().map(|d| Hit { doc_id: d, score: 1.0 }).collect();
    return Ok(Some(self.wrap_with_stride(Box::new(
        VectorIteratorWrapper::new(hits.into_iter())
    ))));
}

#[cfg(feature = "vector_queries")]
if let Query::Vector(kq) = query {
    let hits = self.segment.vector_search(self.schema, kq);
    // ... sort, wrap, return
}
```

Above this point, types are just enum variants (free). Below it, the heavy BKD tree traversal and vector scan live. The cfg gates ensure the feature boundary is visible in code and protected against future changes.

## The Regex Gate

The `fst-regex` crate (finite-state transducer regex engine) is the only dependency specific to regex queries. Gating it behind `regex_queries`:

```toml
regex_queries = ["dep:fst-regex"]
```

Nano doesn't include it; mini does. Combined with the `release-wasm` profile (`opt-level = "z"`, LTO, strip), this dropped nano from 617 KB to 265 KB — then geo/vector gating brought it to the final 245 KB.

## Size Results

All measurements: `release-wasm` profile, `wasm-bindgen --target web`, `gzip -9`.

| Tier | Raw WASM | Gzip | Features |
|------|----------|------|----------|
| **nano** | 768 KB | **245 KB** | text search only |
| **mini** | 3.0 MB | 798 KB | +json, +regex, +geo, +vector, +dsl, +aggs |

## Custom Tier Composition

Users can compose exactly the feature set they need:

```bash
# Text + geo only (no vector, no aggs, no regex)
cargo build --profile release-wasm --target wasm32-unknown-unknown \
  --no-default-features --features "wasm_nano,geo_queries"

# Text + vector only (no geo, no aggs)
cargo build --profile release-wasm --target wasm32-unknown-unknown \
  --no-default-features --features "wasm_nano,vector_queries"

# Full nano build (245 KB gzip)
cargo build --profile release-wasm --target wasm32-unknown-unknown \
  --no-default-features --features wasm_nano
```

## Design Takeaway

For deeply-integrated code in a monolithic crate, **feature flags + LTO** is the pragmatic path:

- Gate at the *execution boundary* (where enum patterns become function calls)
- Not at the *definition boundary* (where types and modules are declared)
- Types are cheap — a 32-byte struct doesn't bloat a binary
- A 4000-LOC BKD tree traversal does — and LTO drops it when nobody calls in

---

## 🏁 Goal Achieved: 1.21 MB → 245 KB

```bash
  1.21 MB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ original
   753 KB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                 Part 2 (−38%)
   617 KB ━━━━━━━━━━━━━━━━━━━━━━━━━                      Part 3 (−49%)
   245 KB ━━━━━━━━━━━                                    Part 4 (−80%) ✅
  <300 KB ━━━━━━━━━━━━━                                  ← goal
```

| Part | Optimization | Nano (gzip) | Reduction |
|------|-------------|-------------|-----------|
| 1 | Typed bindings + `release-wasm` profile | ~900 KB | baseline |
| 2 | Gate `aggs` module | 753 KB | −38% |
| 3 | Eliminate `serde_json` | 617 KB | −49% |
| **4** | **Gate regex + geo + vector** | **245 KB** | **−80%** |

**Target was <300 KB. Final result: 245 KB gzipped.**

A full-featured inverted-index search engine — with BM25 scoring, boolean queries, prefix/wildcard matching, field-level retrieval, and mmap-based segment access — in a package smaller than most JavaScript UI frameworks.

It loads in under 100ms on a 3G connection. It fits in a Service Worker's memory budget with room to spare. And it searches faster than a network round-trip.

---

*Pizza Engine is an embedded search engine written in Rust. The WASM nano tier delivers instant in-browser search without a server. The full engine supports aggregations, vector search, geo queries, graph traversal, and distributed segment access.*
