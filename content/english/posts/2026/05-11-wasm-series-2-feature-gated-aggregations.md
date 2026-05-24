---
title: "Shrinking a Search Engine to Fit in Your Browser — Part 2: Feature-Gated Aggregations"
meta_title: "WASM Size Optimization Series Part 2"
description: "How gating the aggregation subsystem behind a Cargo feature flag cut 40% off the nano WASM binary — from 1.21 MB to 753 KB gzipped."
date: "2026-05-11T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/wasm-series/part2-aggregations.jpeg"
author: "Medcl"
tags: ["WebAssembly", "Rust", "Search Engine", "Performance", "Pizza Engine"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

This is Part 2 of a series on shrinking Pizza Engine's WASM binary from 1.21 MB to 245 KB. In [Part 1](/posts/2026/05-10-wasm-series-1-zero-overhead-bindings/), we designed zero-overhead typed bindings. Now we cut the binary by 40% with a single feature flag.

## The Problem

Not every deployment needs aggregations. A document viewer that mounts a segment and runs keyword queries doesn't need terms buckets, histograms, or significance scoring. But that code was shipping unconditionally — inflating the `.wasm` binary users must download before the first keystroke.

## The Approach

We introduced an `aggs` Cargo feature that gates the entire aggregation subsystem:

```toml
[features]
default = ["aggs"]
aggs = []

wasm_nano = ["wasm", "wasm_panic_hook"]                        # no aggs
wasm_mini = ["wasm_nano", "query_string_parser", "wasm_dsl", "wasm_aggs"]
```

When `aggs` is disabled:

- The `search::aggregator` module is not compiled at all
- `compute_aggregations()` trait method disappears from `StoreReader`
- The executor skips the aggregation pass entirely

When `aggs` is enabled (default, mini, ultra):

- Full aggregation pipeline with 20+ types
- filter/filters/adjacency_matrix with real query evaluation
- top_hits / top_metrics with sort support
- significant_terms with JLH scoring

## What Gets Compiled Out

```bash
src/search/aggregator/
├── mod.rs          ─┐
├── types.rs         │
├── executor.rs      ├─ all gated behind cfg(feature = "aggs")
├── fast.rs          │
└── filter_eval.rs  ─┘
```

The gating is surgical — 6 lines across 5 files:

1. **Module declaration** — `#[cfg(feature = "aggs")] pub mod aggregator;`
2. **Trait method** — `#[cfg(feature = "aggs")] fn compute_aggregations(...)`
3. **Executor call-site** — `#[cfg(feature = "aggs")] { ... }`
4. **WASM glue** — aggs parsing in `wasm/search.rs`
5. **Struct field** — dual-typed: `BTreeMap<String, AggResult>` with aggs, `Option<()>` without

## Size Results

All measurements: `--release`, `wasm-strip`, target `wasm32-unknown-unknown`.

| Tier | Raw WASM | Gzip |
|------|----------|------|
| **nano** (no aggs) | 3.11 MB | **753 KB** |
| **mini** (+aggs, +DSL) | 5.18 MB | 1.24 MB |
| **ultra** (everything) | 5.24 MB | 1.25 MB |

The `aggs` feature accounts for roughly **2 MB raw / 483 KB gzipped** of code. Disabling it keeps nano under 753 KB over the wire.

## Design Decisions

**Why keep `aggregations` in `SearchResult` without the feature?**

Removing the field breaks JSON consumers that expect the key. Instead we define it as `Option<()>` (always `None`, always skipped in serialization) so the struct layout stays compatible.

**Why `default = ["aggs"]`?**

Native builds always want aggregations. Only WASM nano opts out. Default-on means existing dependents don't change their `Cargo.toml`.

**Why not also gate `scripting` inside `aggs`?**

They're orthogonal concerns. `scripting` is independently useful for custom scoring. Keeping them separate lets users combine freely:

- `aggs` without `scripting` — all aggregations except `scripted_metric`
- `scripting` without `aggs` — custom scoring in queries
- both — full power

## Build Commands

```bash
# Nano — smallest, basic search only
cargo build --release --target wasm32-unknown-unknown \
  --no-default-features --features wasm_nano

# Mini — adds DSL + aggregations
cargo build --release --target wasm32-unknown-unknown \
  --no-default-features --features wasm_mini
```

## Progress

```bash
  1.21 MB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ original
   753 KB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                 ← Part 2 (-38%)
  <300 KB ━━━━━━━━━━━━━                                  ← goal
```

| ✅ | Milestone |
|---|---|
| ✅ | Created `aggs` feature flag |
| ✅ | Gated aggregator module (20+ agg types, 5 files) |
| ✅ | Gated `compute_aggregations` trait + executor call-site |
| ✅ | Gated WASM aggregation glue code |
| ✅ | Zero breakage: all 2288 tests pass with and without `aggs` |

**Result: 1.21 MB → 753 KB gzipped (−38%)**

*Next: [Part 3 — Eliminating serde_json](/posts/2026/05-12-wasm-series-3-eliminating-serde-json/)*
