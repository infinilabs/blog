---
title: "True Partial Updates: How Pizza Engine Achieves 870x Faster Document Mutations"
meta_title: "Pizza Engine Partial Document Updates"
description: "Traditional search engines re-index entire documents for any field change. Pizza Engine implements true in-place partial updates — mutating only the fields you touch. A keyword field update takes 1.3 µs vs 1,133 µs for full replacement: 870x faster."
date: "2026-05-26T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/partial-updates.jpeg"
author: "Medcl"
tags: ["Rust", "Search Engine", "Performance", "Pizza Engine", "Real-Time"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

Most search engines treat documents as immutable blobs. Want to change a single field? You have to **delete the entire document and re-index it from scratch** — re-analyzing every text field, rebuilding all postings, reallocating storage. This is how Elasticsearch, Tantivy, and Lucene work under the hood.

Pizza Engine takes a fundamentally different approach.

## The Problem with Full-Document Replacement

Consider a product catalog with 50 fields per document. A price change touches exactly one field — yet traditional engines must:

1. Read the old document from stored fields (disk I/O)
2. Delete the old document (mark tombstone)
3. Re-analyze all 50 fields (CPU-intensive tokenization)
4. Rebuild all posting list entries (memory allocation + insertion)
5. Write the complete new document to a new segment

For a single price update, this is enormously wasteful. The `title`, `description`, `category`, and 47 other fields haven't changed — but they're all re-processed anyway.

## Pizza's In-Place Partial Update

Pizza Engine supports **true partial updates** at the storage layer. When you update a single field:

1. Only that field's old postings are scrubbed
2. Only that field's new value is analyzed and indexed
3. The document is mutated in-place — no full copy
4. All other fields remain untouched

This is not an "update by re-index" wrapper. It's a native operation that walks only the fields you specify.

### The API

Update a single field:

```json
POST /index/_update/42
{
  "doc": {
    "status": "published"
  }
}
```

Or use field-level operations for atomic mutations:

```json
POST /index/_update/42
{
  "operations": {
    "counter": { "increment": 1 },
    "status": { "replace": "active" },
    "tags": { "array_append": ["featured"] }
  }
}
```

### Supported Operations

| Operation | Description |
|-----------|-------------|
| `replace` | Set field to new value (re-indexes) |
| `remove` | Remove field entirely |
| `increment` / `decrement` | Atomic numeric mutation |
| `toggle` | Flip boolean value |
| `array_append` | Append to array field |
| `array_remove` | Remove from array field |
| `array_insert` | Insert at position |
| `array_clear` | Clear array |
| `set_if_absent` | Set only if field doesn't exist |

These are not application-level conveniences that decompose into read-modify-write. They execute as single atomic operations inside the storage engine.

## Performance: The Numbers

We benchmarked mutation operations on a 100K-document index with 4 indexed fields (two text fields with standard analyzer, one keyword, one integer):

| Operation | Latency | Throughput |
|-----------|---------|-----------|
| Full REPLACE (all fields) | 1,133 µs | 883 ops/s |
| **Partial UPDATE (keyword)** | **1.3 µs** | **787,000 ops/s** |
| Partial UPDATE (text, re-analyzed) | 117 µs | 8,600 ops/s |
| Partial UPDATE (integer increment) | 64 µs | 15,500 ops/s |
| Partial UPDATE (3 fields combined) | 185 µs | 5,400 ops/s |
| DELETE | 0.02 µs | 41,700,000 ops/s |

**A keyword field update is 870x faster than full document replacement.**

Even a text field update — which requires re-tokenization — is still **10x faster** than a full replace, because only one field is processed instead of all four.

The cost scales with what you touch, not with document size. A 50-field document updated on one keyword field costs the same 1.3 µs as a 4-field document.

## Why This Matters

### Real-Time Analytics

Counters, view counts, scores, timestamps — these change constantly. With partial updates, you can increment a counter at 787K ops/sec without disrupting the rest of the document's index state. No background merges. No refresh delays.

### E-Commerce

Price updates, stock levels, availability flags — all keyword or numeric fields that update millions of times per day. At 1.3 µs per update, a single thread can handle your entire catalog's price feed in real-time.

### Content Management

Publishing workflows where `status` transitions from `draft` → `review` → `published`. A single keyword field change that shouldn't trigger expensive full-text re-analysis of the 10,000-word document body.

### IoT & Streaming Data

Sensor readings that append to arrays or update latest-value fields. The `array_append` and `increment` operations are purpose-built for append-heavy patterns without read-modify-write overhead.

## How It Works Internally

Pizza's mutable segment stores documents in a chunked arena with direct slot addressing. When a partial update arrives:

```text
1. Locate document by ID → O(1) lookup via doc_id → slot map
2. For each field in the update:
   a. Snapshot the current field value (single field, not full doc)
   b. Compute the outcome (Replace? Remove? Increment result?)
   c. Scrub stale postings for THAT field only
   d. Re-index the new value via the field's analyzer
   e. Mutate the stored document in-place (zero-copy)
3. Bump the slot version → snapshot isolation preserved
```

The key insight: **no full-document clone ever happens**. Each field is processed independently, and the document's arena slot is mutated in-place.

This is fundamentally different from Lucene's immutable-segment architecture, where _any_ document change — even toggling a boolean — requires allocating a new document in a new segment and tombstoning the old one.

## Compared to Elasticsearch's _update

Elasticsearch's `_update` API looks similar on the surface. Under the hood it:

1. Fetches the entire `_source` from stored fields (disk read)
2. Applies your patch in memory (JSON merge)
3. Re-indexes the **entire** document — all fields re-analyzed
4. Writes to a new Lucene segment
5. Old document becomes a tombstone (cleaned up during merge)

This means Elasticsearch's "partial update" has **the same cost as a full replace** — it's a convenience wrapper, not a performance optimization. The document size doesn't matter; the engine always does full work.

Pizza's partial update is a true sub-document operation with cost proportional to the fields touched.

## Real-Time Visibility

Updates are immediately visible to new queries after flush — no segment merge required, no `refresh_interval` to wait for:

- Write partial update: **1.3 µs**
- Flush (make searchable): **< 1 µs**
- Total write-to-searchable latency: **< 3 µs**

Compare this to Elasticsearch's default 1-second refresh interval — that's a 300,000x difference in visibility latency.

## Conclusion

Partial document updates aren't just a nice API convenience — they're a **fundamental performance feature** when your workload involves frequent mutations to specific fields.

Pizza Engine's architecture makes this possible by treating documents as mutable, field-addressable structures rather than immutable blobs. The mutable layer handles high-frequency updates efficiently, while background compaction builds read-optimized immutable segments for query performance.

The 870x speedup over full replacement means workloads that were previously impractical — real-time counters, high-frequency price feeds, IoT sensor streams — can now run directly on the search engine without external caching layers or update queues.

---

*The benchmarks in this post are reproducible via `cargo run --release --bin bench_mutations` in the Pizza Engine repository.*
