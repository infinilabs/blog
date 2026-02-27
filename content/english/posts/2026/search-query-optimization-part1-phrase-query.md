---
title: "Search Query Optimization Part 1: Phrase Query — From 4ms to 386μs"
meta_title: "Search Query Optimization Part 1: Phrase Query — From 4ms to 386μs"
description: "How we optimized Pizza Engine's phrase queries from 10x slower than Lucene to 3x faster, using rarest-term-first lead, galloping search, and streaming visitor architecture."
date: "2026-02-26T10:00:00+08:00"
categories: ["Engineering", "Performance"]
image: "/images/posts/2026/search-query-optimization-part1.png"
author: "Medcl"
tags: ["Pizza Engine","FIRE", "Search", "Performance", "Rust", "Phrase Query"]
lang: "en"
category: "Engineering"
subcategory: "Performance"
draft: false
---

*This is Part 1 of a series on how we optimized Pizza Engine's search queries to outperform Lucene. In this post, we tackle phrase queries — taking them from 10x slower than Lucene to 3x faster.*

## The Setup

Pizza Engine stores its immutable index in a structure called `CompactSegment` — a per-field inverted index with pre-computed BM25 scores. Each term's posting list contains entries sorted by `doc_id`:

```rust
struct PostingEntry {
    doc_id: u32,
    score: f32,           // pre-computed BM25
    term_frequency: u32,
    pos_offset: u32,      // offset into positions arena
    pos_len: u32,         // number of positions
}
```

Token positions for all entries in a field are stored in a shared arena (`Vec<u32>`), so each entry just records its offset and length — no per-entry `Vec<u32>` allocation.

## The Problem

Our initial phrase query implementation was straightforward:

1. Look up each term in the phrase
2. Find documents that contain **all** terms (intersection)
3. For each candidate, verify that the terms appear at **consecutive positions**
4. Return matching documents with BM25 scores

The bottleneck was immediately obvious when we benchmarked against Lucene on 5 million Wikipedia articles:

| Query | Pizza (μs) | Lucene-BP (μs) | Ratio |
|-------|--------:|--------:|------:|
| `"new york population"` | 8,241 | 4,403 | 1.87x slower |
| `"immigration to mexico"` | 6,102 | 910 | 6.71x slower |
| **Median (all phrase)** | **4,057** | **1,121** | **3.62x slower** |

The profile showed three problems:

1. **Leading with the wrong term.** We were iterating the first term and checking others. For `"the globe newspaper"`, "the" has ~4M postings while "globe" has ~2K. We were doing 4M iterations instead of 2K.

2. **Linear scanning for intersection.** After picking a candidate `doc_id` from the driver, we used `binary_search` on the other posting lists. This is O(log n) per probe but ignores the fact that both lists are sorted — we throw away the cursor position each time.

3. **Materializing everything.** We allocated a `Vec<Hit>` per term, intersected them, *then* checked positions. For common terms, this means allocating millions of `Hit` structs that get discarded.

## Fix 1: Rarest-Term-First Lead

The simplest and highest-impact fix: sort terms by `doc_frequency` and let the **rarest term drive** the intersection.

```rust
// Sort term postings by doc_frequency ascending
term_postings.sort_unstable_by_key(|tp| tp.doc_frequency);

let driver = &term_postings[0];  // rarest term drives
let others = &term_postings[1..];
```

For `"the globe newspaper"`:
- Before: driver = "the" (4,012,345 postings), probing "globe" (2,103) and "newspaper" (18,432)
- After: driver = "globe" (2,103 postings), probing "newspaper" (18,432) and "the" (4,012,345)

This alone turned many queries from millions of iterations into thousands.

## Fix 2: Galloping Search

Since both the driver and the probe lists are sorted by `doc_id`, we can maintain a cursor and use **galloping (exponential) search** instead of binary search:

```rust
fn gallop_search(
    entries: &[PostingEntry],
    target_doc_id: u32,
    from: usize,  // cursor — never go backwards
) -> Option<usize> {
    let cur = entries[from].doc_id;
    if cur == target_doc_id { return Some(from); }
    if cur > target_doc_id  { return None; }

    // Exponential search: double the step until we overshoot
    let mut step = 1;
    let mut lo = from;
    let mut hi = from + step;
    while hi < len && entries[hi].doc_id < target_doc_id {
        lo = hi;
        step *= 2;
        hi = (from + step).min(len);
    }

    // Binary search within the narrow [lo, hi) window
    entries[lo..hi].binary_search_by_key(&target_doc_id, |e| e.doc_id)
}
```

The key insight: galloping is O(log d) where d is the **distance** from the cursor to the target. When the driver has few postings and the probe list is dense, most probes land very close to the previous cursor position, making galloping nearly O(1) per probe.

For each candidate doc in the driver:
```rust
for entry in &driver.entries {
    let doc_id = entry.doc_id;

    // Gallop through each other term's postings
    for (j, other) in others.iter().enumerate() {
        match gallop_search(&other.entries, doc_id, cursors[j]) {
            Some(idx) => {
                cursors[j] = idx + 1;  // advance cursor
                // found — continue to next term
            }
            None => continue 'outer,  // not found — skip this doc
        }
    }

    // All terms present — now verify phrase positions...
}
```

## Fix 3: Streaming Visitor Architecture

Instead of building a complete `Vec<Hit>` and returning it, we switched to a **visitor pattern** that streams results one at a time:

```rust
fn search_phrase_visit<F>(
    &self, ctx, schema, field_name, query_text,
    visitor: &mut F,  // called with each matching Hit
) where F: FnMut(Hit) -> bool  // return false to stop early
```

The caller (a `TopKCollector`) returns `false` once it has enough results and the heap threshold is high enough that no more documents in this segment can enter the top-K. This enables **early termination** — for TOP_10 queries, we often stop after examining a fraction of the matches.

## Fix 4: Pre-Allocated Scratch Buffers

Position checking requires comparing position arrays across terms. Instead of allocating temporary vectors per document, we pre-allocate scratch buffers:

```rust
// Pre-allocate position scratch buffers (reused across docs)
let mut pos_buffers: Vec<Vec<u32>> = (0..term_postings.len())
    .map(|_| Vec::with_capacity(64))
    .collect();

for candidate_doc in driver {
    // Reuse buffers — clear but keep allocation
    for buf in &mut pos_buffers { buf.clear(); }

    // Fill positions from the arena
    for (i, tp) in term_postings.iter().enumerate() {
        let positions = field_index.positions_of(&tp.entries[idx]);
        pos_buffers[i].extend_from_slice(positions);
    }

    // Check consecutive positions...
}
```

## Results

After all four optimizations, the combined effect was dramatic:

| Metric | Before | After | Speedup |
|--------|-------:|------:|--------:|
| Median phrase (TOP_10) | 4,057μs | 386μs | **10.5x** |
| vs Lucene-BP | 3.62x slower | **0.34x** (2.9x faster) | — |
| "immigration to mexico" | 6,102μs | 427μs | 14.3x |
| "new york population" | 8,241μs | 1,852μs | 4.5x |

The optimization stack works multiplicatively:

1. **Rarest-term lead** eliminates ~99% of iterations for stopword-heavy phrases
2. **Galloping** makes each remaining probe O(log d) instead of O(log n)
3. **Streaming visitor** enables early termination for TOP_K
4. **Scratch buffers** eliminate per-document allocation

None of these require complex data structures like skip lists or block-based posting formats. They're pure algorithmic improvements on the existing sorted posting lists.

## What's Next

Phrase queries are now fast, but we noticed that **COUNT queries** — which need to count all matching documents without scoring — were still bottlenecked by our scoring-oriented data structures. In Part 2, we'll introduce cache-efficient counting with parallel `u32` arrays.

---

*Part 2: [COUNT Query — The Art of Not Scoring](/posts/2026/search-query-optimization-part2-count-query/)*
