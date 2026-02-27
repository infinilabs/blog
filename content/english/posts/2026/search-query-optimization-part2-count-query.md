---
title: "Search Query Optimization Part 2: COUNT Query — The Art of Not Scoring"
meta_title: "Search Query Optimization Part 2: COUNT Query — The Art of Not Scoring"
description: "How we made Pizza Engine's COUNT queries 18x faster with O(1) term count, cache-efficient u32 arrays, and bitset union — from 13x slower to 1.4x faster than Lucene-BP."
date: "2026-02-26T12:00:00+08:00"
categories: ["Engineering", "Performance"]
image: "/images/posts/2026/search-query-optimization-part2.png"
author: "Medcl"
tags: ["Pizza Engine","FIRE", "Search", "Performance", "Rust", "COUNT Query", "Cache Efficiency"]
lang: "en"
category: "Engineering"
subcategory: "Performance"
draft: false
---

*This is Part 2 of a series on optimizing Pizza Engine's search queries. In Part 1, we tackled phrase queries. Here, we address COUNT queries — which need to count matching documents without scoring — and discover that the key optimization is about CPU cache lines, not algorithms.*

## The Problem

COUNT queries (`SELECT COUNT(*) WHERE ...`) only need a number — no scores, no documents, no sorting. Yet our initial implementation ran the full `search()` pipeline: build `Vec<Hit>` with scores, intersect, collect, then just return `.len()`.

For a single-term COUNT, this meant iterating 200K+ `PostingEntry` structs (20 bytes each) and allocating a `Vec<Hit>` just to call `.len()`. For intersections, it was worse — multiple huge vectors allocated and merged, only to count the result.

The benchmark told the story:

| Category | Pizza (μs) | Lucene-BP (μs) | P/BP |
|----------|--------:|--------:|-----:|
| Single term | 113,174 | 18 | 6,288x slower |
| Intersection | 1,062 | 640 | 1.66x slower |
| Union | 2,541 | 2,191 | 1.16x slower |
| **OVERALL** | **~20,000** | **1,506** | **13.3x slower** |

Single-term COUNT was 6,000x slower than Lucene. Clearly we were doing something fundamentally wrong.

## Insight: O(1) Term Count

For a single-term COUNT, the answer is already stored:

```rust
struct TermPostings {
    doc_frequency: u32,  // ← this IS the count
    max_score: f32,
    entries: Vec<PostingEntry>,
}
```

The `doc_frequency` field records exactly how many documents contain this term. We don't need to iterate the posting list at all:

```rust
fn count_query(&self, query: &Query) -> u32 {
    match query {
        Query::Term(tq) => {
            // O(1): just return the stored doc_frequency
            let fi = self.field_indices.get(field)?;
            let tid = convert_to_term_id(term, &fi.term_dict)?;
            fi.postings[tid].doc_frequency
        }
        // ...
    }
}
```

**Single-term COUNT: 113,174μs → 32μs.** That's 3,537x faster, from a one-line change.

## The Cache Problem

For intersection COUNT (e.g., `+new +york`), we still need to intersect two posting lists. Our existing `merge_and()` worked on `Vec<Hit>` which meant iterating `PostingEntry` arrays:

```text
PostingEntry: [doc_id(4B) | score(4B) | tf(4B) | pos_off(4B) | pos_len(4B)]
              └─────────────── 20 bytes per entry ───────────────┘
```

A CPU cache line is 64 bytes — it fits **3.2 PostingEntry** structs. But for COUNT, we only need the `doc_id` field. The other 16 bytes per entry (score, tf, positions) are dead weight that pollutes the cache.

For a term with 200K postings: 200K × 20B = 3.8 MB of data touched, but only 200K × 4B = 0.78 MB is useful. We're wasting **80%** of our cache bandwidth.

## Fix: Parallel u32 Doc-ID Array

We added a parallel array that stores just the doc_ids:

```rust
struct TermPostings {
    doc_frequency: u32,
    max_score: f32,
    entries: Vec<PostingEntry>,    // 20 B/entry — for scoring
    doc_ids: Vec<u32>,             // 4 B/entry — for counting
}
```

The `doc_ids` array is built alongside `entries` during indexing and maintains the same sorted order. It's a classic **Structure of Arrays (SoA)** transformation — instead of an Array of Structs where each struct has a `doc_id`, we store all `doc_id` values contiguously.

Now a cache line fits **16 doc_ids** instead of 3.2. The working set for 200K postings drops from 3.8 MB to 0.78 MB — well within L2 cache.

## Galloping Intersection on u32 Arrays

With the compact arrays, we wrote a dedicated intersection routine:

```rust
fn count_sorted_intersection_u32(slices: &[&[u32]]) -> u32 {
    // Sort by length — rarest term drives
    let sorted = sort_by_length(slices);
    let driver = sorted[0];
    let others = &sorted[1..];
    let mut cursors = vec![0usize; others.len()];
    let mut count = 0u32;

    'outer: for &doc_id in driver {
        for (j, other) in others.iter().enumerate() {
            match gallop_search_u32(other, doc_id, cursors[j]) {
                Some(idx) => cursors[j] = idx + 1,
                None => {
                    if cursors[j] >= other.len() {
                        return count;  // exhausted → done
                    }
                    continue 'outer;
                }
            }
        }
        count += 1;
    }
    count
}
```

Same galloping algorithm as Part 1, but on `u32` slices instead of `PostingEntry` slices. The difference is entirely in cache behavior.

## Bitset Union for OR Queries

For union COUNT (e.g., `battle of the bulge`), we need to count **distinct** doc_ids across multiple posting lists. A hash set would work but has poor cache behavior. Instead, we use a **self-sizing bitset**:

```rust
fn count_union_bitset_u32(slices: &[&[u32]]) -> u32 {
    // Find the max doc_id to size the bitset
    let max_id = slices.iter()
        .filter_map(|s| s.last())
        .max()
        .unwrap();

    // Allocate bitset: 1 bit per possible doc_id
    let num_words = (max_id as usize / 64) + 1;
    let mut bits = vec![0u64; num_words];

    // Set bits for all doc_ids
    for slice in slices {
        for &doc_id in *slice {
            bits[doc_id as usize / 64] |= 1u64 << (doc_id % 64);
        }
    }

    // popcount to get the union size
    bits.iter().map(|w| w.count_ones() as u32).sum()
}
```

Modern CPUs have a hardware `POPCNT` instruction that counts set bits in a single cycle. The bitset approach is both simpler and faster than hash-based deduplication.

## Phrase-Aware Medium Path

Some COUNT queries involve phrases within boolean expressions, like `+"the who" +uk`. A phrase can't be resolved to a single posting list — it requires position checking. But we can still optimize:

1. **Decompose the phrase** into its constituent terms
2. **Intersect the u32 arrays** first (fast, cache-efficient)
3. Only for documents that pass the intersection, do the **expensive position check**

This is a "medium path" between the fast u32-only path and the full materialized search:

```rust
// Medium path: phrases mixed with terms
// Step 1: Get u32 doc_ids from all simple terms
// Step 2: Intersect u32 arrays (fast)
// Step 3: For surviving docs, verify phrase positions (slow but rare)
```

## Results

| Category | Before (μs) | After (μs) | P/BP before | P/BP after |
|----------|--------:|--------:|-----:|-----:|
| Single term | 113,174 | 21 | 6,288x | **1.17x** |
| Intersection | 1,062 | 538 | 1.66x | **0.84x** |
| Union | 2,541 | 1,637 | 1.16x | **0.75x** |
| Phrase | 1,865 | 1,113 | 1.10x | **0.66x** |
| **OVERALL** | **~20,000** | **1,094** | **13.3x** | **0.73x** |

The COUNT query went from **13.3x slower** than Lucene-BP to **0.73x** (1.37x faster). The key wins:

- **O(1) single-term**: eliminated iteration entirely
- **u32 cache-efficient arrays**: 5x less memory bandwidth for intersections
- **Bitset union**: hardware POPCNT for OR queries
- **Phrase-aware medium path**: position checks only for docs that pass u32 intersection

## The Lesson: Data Layout Matters More Than Algorithms

The galloping algorithm was identical between Part 1 (PostingEntry) and Part 2 (u32). The only change was **data layout** — pulling `doc_id` into its own array. This single SoA transformation turned a 1.66x deficit into a 0.84x advantage.

In search engine optimization, cache efficiency often dominates algorithmic complexity. A theoretically optimal algorithm on poorly laid out data will lose to a simpler algorithm on cache-friendly data.

## What's Next

With COUNT fast and phrase queries fast, one bottleneck remained: **TOP_K intersection queries** containing stopwords. Queries like `+the +incredibles` were still 13x slower than Lucene-BP because we were materializing enormous `Vec<Hit>` for common terms. In Part 3, we'll introduce two-phase streaming conjunction to eliminate this last bottleneck.

---

*Part 1: [Phrase Query — From 4ms to 386μs](/posts/2026/search-query-optimization-part1-phrase-query/)*

*Part 3: [Intersection Query — Two-Phase Streaming Conjunction](/posts/2026/search-query-optimization-part3-intersection-query/)*
