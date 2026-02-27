---
title: "Introducing Pizza Engine — A Next-Generation Search Engine Written in Rust"
meta_title: "Introducing Pizza Engine — A Next-Generation Search Engine Written in Rust"
description: "Pizza Engine (code name FIRE) is a lightweight, embeddable, real-time search engine library built from scratch in Rust. With a share-nothing, no_std architecture, it delivers sub-millisecond queries and outperforms Lucene on standard benchmarks."
date: "2026-02-25T10:00:00+08:00"
categories: ["Engineering", "Announcement"]
image: "/images/posts/2026/introducing-pizza-engine.png"
author: "Medcl"
tags: ["FIRE", "Pizza Engine", "Search", "Rust", "Announcement"]
lang: "en"
category: "Engineering"
subcategory: "Search"
draft: false
---

What if you could build a search engine that runs on a single CPU core, compiles to WebAssembly, requires zero operating system dependencies — and still beats Lucene?

That's the question that led to **Pizza Engine**.

## What Is Pizza Engine?

Pizza Engine, code name **FIRE** (**F**ast **I**ndexing and **R**etrieval **E**ngine), is a lightweight, embeddable, real-time search engine library written entirely in Rust. It's designed from the ground up as a pure computation library — no threads, no locks, no I/O, no runtime. Just data structures and algorithms.

```rust
let mut builder = EngineBuilder::new();
builder.set_schema(schema);
builder.set_data_store(LayeredStore::new());
let engine = builder.build().unwrap();
engine.start();

// Documents are searchable immediately after write
let mut writer = engine.acquire_writer();
writer.create_document(doc).unwrap();
writer.flush().unwrap();

// Query
let snapshot = engine.create_snapshot();
let searcher = engine.acquire_searcher();
let result = searcher.parse_and_query(&query, &snapshot).unwrap();
```

That's it. No cluster to set up, no JVM to tune, no configuration files to wrestle with.

## Why Build Another Search Engine?

The short answer: because existing engines weren't designed for where computing is going.

Modern search engines — Lucene, Elasticsearch, Solr — are incredible pieces of engineering. But they share assumptions that made sense in 2010 and are increasingly at odds with today's hardware reality:

- **Thread-shared indexes** require locks, atomics, and careful synchronization. As core counts grow, contention becomes the bottleneck — not computation.
- **JVM garbage collection** introduces unpredictable latency spikes. Search engines need deterministic sub-millisecond response times.
- **Refresh intervals** mean documents aren't searchable until the next "refresh" cycle — typically once per second. That's an eternity for real-time applications.
- **Heavyweight deployment** ties search to server infrastructure. You can't embed Lucene in a browser, an edge device, or a smart appliance.

Pizza Engine starts from different assumptions:

| | Traditional Engines | Pizza Engine |
|---|---|---|
| Concurrency | Thread pools + shared indexes | One engine per core, share-nothing |
| Memory | JVM heap + GC | Arena allocation, zero-copy, no GC |
| Latency | Refresh interval (1s default) | True real-time — write and search instantly |
| Portability | JVM / native binary | `#![no_std]` — bare metal, WASM, anywhere |
| Dependencies | OS threads, file I/O, networking | None. Pure computation library. |

## Design Principles

### Share-Nothing: One Engine Per Core

Pizza Engine is designed so that each CPU core runs its own independent engine instance. There are no mutexes, no `std::sync`, no cross-thread sharing. This isn't a limitation — it's a deliberate architectural choice that eliminates an entire class of performance problems.

When you remove lock contention from the picture, performance scales linearly with core count. A 128-core machine runs 128 independent engines, each operating at full speed with zero interference.

### `#![no_std]`: No OS, No Problem

The entire engine compiles under Rust's `#![no_std]` mode. It makes no system calls, spawns no threads, and performs no I/O. This means Pizza Engine can run:

- In a **WebAssembly** sandbox inside a browser
- On **bare-metal** embedded devices
- Inside a **serverless function** with minimal cold start
- As a **library** embedded in any application

### True Real-Time: Write and Search Instantly

There is no refresh interval. Documents become searchable the moment they are written. Pizza Engine achieves this through snapshot isolation — each read operation sees a consistent point-in-time view, while writes proceed without blocking reads.

### Dual-Layer Storage Architecture

The engine uses a two-layer storage design that combines the best of mutable and immutable data structures:

```text
Write Request ──► Mutable Layer ──freeze/commit──► Immutable Layer
                       ▲                                │
                       │                                ▼
                  Real-time CRUD                  Read-only, compact
                  In-memory structures            Pre-computed BM25
                  Snapshot isolation               Block-Max WAND optimization
```

- **Mutable Layer** — Handles real-time writes using HashMap and RoaringBitmap. BM25 scores are computed at query time. Fast to write, fast to read for small datasets.
- **Immutable Layer** — Built from frozen mutable epochs. Uses FST term dictionaries, hierarchical Segment/Block posting lists, and pre-computed BM25 scores. Optimized for large-scale read performance.

Queries search both layers simultaneously and merge results.

### Arena-Based Memory Management

Instead of per-object heap allocation, immutable segments store all their data in contiguous Arena memory blocks. This provides:

- **Cache-friendly access** — sequential memory layout, SIMD-friendly
- **Zero-copy reads** — data is accessed in-place, no deserialization
- **Instant deallocation** — when a segment is garbage collected, its entire Arena is released in one operation. No destructors, no fragmentation.

## What Can It Do?

Pizza Engine supports the query types you'd expect from a modern search engine:

| Field Type | Index Structure | Queries |
|---|---|---|
| Text | Inverted Index | term, match, phrase, prefix, wildcard, fuzzy, regex |
| Keyword | Inverted Index | term, prefix, wildcard |
| Boolean | BitSet | term |
| Numeric | BKD Tree | term, range |
| Object | (recursive) | Flattened dot-path access |

It supports Lucene-style query strings (`title:pizza AND status:published`), JSON query DSL (term, match, phrase, bool, range, and more), BM25 scoring with configurable parameters, and query explanation for debugging.

## How Fast Is It?

We benchmarked Pizza Engine against Lucene, Lucene with Block-max WAND and doc-Id reordering (Lucene-BP), and Tantivy — using the [search-benchmark-game](https://github.com/tantivy-search/search-benchmark-game) methodology: **5 million Wikipedia documents, 902 real-world queries**.

Here are the headline numbers (lower is better, ratio < 1.0 means Pizza is faster):

| Category | Pizza (μs) | vs Lucene-BP |
|---|---|---|
| **TOP_10 Overall** | 223 | **3.5x faster** |
| **TOP_100 Overall** | 420 | **2.7x faster** |
| **COUNT Overall** | 1,123 | **1.4x faster** |

The engine is faster than Lucene-BP across every major query category — single-term, phrase, conjunction, disjunction, and COUNT. And it does so while maintaining **100% query correctness**: zero mismatches across all 8,118 result comparisons.

We wrote a [4-part series](/posts/2026/search-query-optimization-part1-phrase-query/) detailing how we got there:

1. [Phrase Query — From 4ms to 386μs](/posts/2026/search-query-optimization-part1-phrase-query/)
2. [COUNT Query — The Art of Not Scoring](/posts/2026/search-query-optimization-part2-count-query/)
3. [Intersection Query — Two-Phase Streaming Conjunction](/posts/2026/search-query-optimization-part3-intersection-query/)
4. [Final Scorecard — Beating Lucene](/posts/2026/search-query-optimization-part4-final-scorecard/)

## What's Next?

Pizza Engine is still in active development. There's a long list of things we're working on:

- **Distributed coordination** — Scaling from per-core engines to multi-node clusters
- **Tiered storage** — Arena memory → NVMe → S3 for cost-efficient cold data
- **Advanced ranking** — Learning-to-rank integration, custom scoring functions
- **Aggregations** — Facets, histograms, and analytics queries
- **More analysis** — CJK tokenizers, stemming, synonyms, and language-specific pipelines

## When Can I Try It?

Pizza Engine is **not open source yet**. We're polishing the API, hardening edge cases, and building out the distributed layer before we open the doors.

**Stay tuned.** When we're ready, you'll be the first to know.

If you're interested in search engine internals, Rust performance engineering, or just want to follow along — check out the [optimization blog series](/posts/2026/search-query-optimization-part1-phrase-query/) and watch this space.

---

*Pizza Engine is built by the team at [INFINI Labs](https://infinilabs.com). We build tools for search and data infrastructure.*
