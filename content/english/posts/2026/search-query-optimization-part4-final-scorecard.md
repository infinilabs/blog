---
title: "Search Query Optimization Part 4: Beating Lucene — Final Scorecard"
meta_title: "Search Query Optimization Part 4: Beating Lucene — Final Scorecard"
description: "Pizza Engine vs Lucene benchmark results: 3.5x faster on TOP_10, 2.7x on TOP_100, 1.4x on COUNT — with 100% query correctness across 902 queries and 5M Wikipedia documents."
date: "2026-02-27T12:00:00+08:00"
categories: ["Engineering", "Performance"]
image: "/images/posts/2026/search-query-optimization-part4.png"
author: "Medcl"
tags: ["Pizza Engine","FIRE", "Search", "Performance", "Rust", "Benchmark", "Lucene"]
lang: "en"
category: "Engineering"
subcategory: "Performance"
draft: false
---


*This is the final part of our series on optimizing Pizza Engine's search queries. We started with phrase queries 10x slower than Lucene and ended with a system that is 2–12x faster across all query types. Here's the complete picture.*

## The Benchmark

We benchmarked against the [search-benchmark-game](https://github.com/tantivy-search/search-benchmark-game) framework with four engines on 5 million Wikipedia articles (902 unique queries). The full benchmark source and raw results are available at [infinilabs/pizza-engine-benchmark](https://github.com/infinilabs/pizza-engine-benchmark/tree/f6a00843bca6dc1bfb42297758a52c969fae4a2d), and you can explore the interactive results at [engine-benchmarks.pizza.rs](https://engine-benchmarks.pizza.rs/).

Engines tested:

- **Lucene 9.9.2** — the standard reference
- **Lucene 9.9.2-BP** — Lucene with Bipartite graph Partitioning doc-ID reordering (state-of-the-art)
- **Tantivy 0.22** — Rust search library
- **Pizza Engine 0.1** — our engine

Three query modes:
- **TOP_10** — return 10 highest-scoring documents
- **TOP_100** — return 100 highest-scoring documents
- **COUNT** — return total matching document count (no scoring)

## Final Numbers

### TOP_10 (median μs, lower is better)

| Category | Lucene | Lucene-BP | Tantivy | **Pizza** | P/BP |
|----------|-------:|----------:|--------:|--------:|-----:|
| Intersection | 905 | 573 | 1,811 | **44** | **0.08x** |
| Phrase | 1,673 | 1,145 | 1,684 | **379** | **0.33x** |
| Union | 1,009 | 614 | 1,674 | **247** | **0.40x** |
| Single term | 877 | 675 | 1,589 | **113** | **0.17x** |
| **OVERALL** | **1,195** | **777** | **1,723** | **223** | **0.29x** |

Pizza is **3.5x faster** than Lucene-BP on TOP_10, and **12.5x faster** on intersection queries.

### TOP_100 (median μs)

| Category | Lucene | Lucene-BP | Tantivy | **Pizza** | P/BP |
|----------|-------:|----------:|--------:|--------:|-----:|
| Intersection | 992 | 734 | 1,802 | **152** | **0.21x** |
| Phrase | 1,934 | 1,573 | 1,690 | **859** | **0.55x** |
| Union | 1,280 | 1,043 | 2,694 | **250** | **0.24x** |
| Single term | 2,905 | 1,767 | 2,507 | **117** | **0.07x** |
| **OVERALL** | **1,403** | **1,117** | **2,062** | **420** | **0.38x** |

Pizza is **2.7x faster** than Lucene-BP on TOP_100.

### COUNT (median μs)

| Category | Lucene | Lucene-BP | Tantivy | **Pizza** | P/BP |
|----------|-------:|----------:|--------:|--------:|-----:|
| Intersection | 910 | 647 | 1,544 | **553** | **0.85x** |
| Phrase | 2,026 | 1,714 | 1,656 | **1,151** | **0.67x** |
| Union | 2,373 | 2,191 | 2,684 | **1,671** | **0.76x** |
| Single term | 16 | 21 | 11 | 27 | 1.29x |
| **OVERALL** | **1,767** | **1,515** | **1,959** | **1,123** | **0.74x** |

Pizza is **1.35x faster** than Lucene-BP on COUNT.

### Correctness

902 queries × 3 commands × 3 engines compared against Lucene as reference: **0 mismatches across 8,118 comparisons.**

## The Optimization Journey

Here's how each optimization moved the needle, shown as P/BP ratio (lower is better):

```text
Starting point (naive implementation):
  TOP_10:  ~2.0x slower than BP
  COUNT:   ~13x slower than BP

After Part 1 — Phrase query optimization:
  Phrase:  3.6x → 0.34x    (10.5x improvement)
  TOP_10:  2.0x → 0.38x

After Part 2 — COUNT query optimization:
  COUNT:   13.3x → 0.73x   (18x improvement)
  Single:  6288x → 1.17x   (5,375x improvement)

After Part 3 — Intersection streaming:
  Inter:   0.27x → 0.08x   (3.4x improvement)
  TOP_10:  0.38x → 0.29x
  TOP_100: 0.46x → 0.38x
```

## What Made It Possible

### Pre-computed BM25

The `CompactSegment` pre-computes BM25 scores at index time. Each posting entry stores the final score, so query-time scoring is a simple lookup. This eliminates the formula evaluation that Lucene and Tantivy perform per-document at query time.

The trade-off is larger posting entries (20 bytes vs ~8 for doc_id + frequency) and no support for query-time boosting or custom similarity. For the standard BM25 use case, it's a clear win.

### Structure of Arrays (SoA)

The parallel `doc_ids: Vec<u32>` array alongside `entries: Vec<PostingEntry>` enables choosing the right data layout for each operation:

- **Scoring (TOP_K):** iterate `entries` (20B/entry) — need scores
- **Counting (COUNT):** iterate `doc_ids` (4B/entry) — 5x less cache pressure
- **Existence check:** gallop on `doc_ids` — cache-efficient Phase 1

One index, two layouts, tailored to the access pattern.

### Streaming Visitor Pattern

The visitor architecture (`FnMut(Hit) -> bool`) enables early termination at every level:

```text
LayeredStore::search()
  for each segment:
    segment.search_visit(visitor)
      for each matching doc:
        visitor(hit)
          → TopKCollector.collect_one(hit)
          → return false if heap is full and threshold unreachable
```

This is fundamentally different from the "materialize then sort" approach. In the best case, a TOP_10 query stops after examining a few hundred documents instead of millions.

### Galloping Everywhere

We use galloping (exponential + binary) search in four distinct contexts:

1. **Phrase position verification** — gallop through position lists
2. **PostingEntry intersection** — gallop through `entries` for scored results
3. **u32 intersection for COUNT** — gallop through `doc_ids`
4. **Two-phase conjunction** — gallop through `doc_ids` for existence, `entries` for score

The galloping search is O(log d) where d is the distance from the cursor to the target. Since both lists are sorted and we advance cursors monotonically, consecutive probes tend to be close together, making galloping nearly O(1) in practice.

## What We Didn't Need

### Block-max WAND

The standard technique for TOP_K optimization in Lucene. We achieved better results on intersection queries through rarest-term-first + streaming, with zero index format changes.

Block-max WAND would still help for **disjunctive (OR) TOP_K** queries where you need to find high-scoring documents across multiple large posting lists. Our union TOP_K (0.40x BP on TOP_10) is already fast, but there's room for improvement on the worst cases.

### BP Doc-ID Reordering

Lucene-BP reorders document IDs so that similar documents have adjacent IDs, improving block-level skip rates. This is an index-time optimization that benefits all query types.

We beat Lucene-BP without reordering because our constant factors are smaller — no JVM overhead, pre-computed scores, cache-efficient data layouts. If we added reordering, the gap would widen further.

### Skip Lists

Traditional search engines use skip lists for fast posting list traversal. Our posting lists are simple sorted arrays with galloping search, which has similar asymptotic behavior but better cache locality (no pointer chasing).

## Remaining Weaknesses

Honest accounting of where we're still slow:

1. **COUNT single term: 1.29x BP.** Our O(1) lookup returns 27μs vs BP's 21μs — likely just measurement noise at this scale, but Lucene's codec is slightly more efficient for metadata access.

2. **TOP_10 union worst cases: 3–6x BP.** Queries like `the oregonian newspaper` (OR) are 6.3x slower than BP. These benefit from Block-max WAND's threshold pruning on disjunctive queries — an area where our current approach doesn't help.

3. **COUNT phrase: 0.67x BP.** Good but not great. Position checking is inherently expensive and our approach still iterates all candidate documents.

## Architecture Summary

```text
┌─────────────────────────────────────────────────┐
│  CompactSegment (per segment, in-memory)          │
│                                                    │
│  FieldIndex                                        │
│  ├─ ImmutableFstDict (term → term_id)             │
│  ├─ postings: Vec<TermPostings>                   │
│  │   ├─ entries: Vec<PostingEntry>  (20B/entry)   │
│  │   │   doc_id | score | tf | pos_off | pos_len  │
│  │   └─ doc_ids: Vec<u32>          (4B/entry)     │
│  │       Parallel SoA for cache-efficient ops      │
│  └─ positions_arena: Vec<u32>                      │
│      Shared arena for all position data            │
│                                                    │
│  Query Dispatch                                    │
│  ├─ Term     → O(1) count / streaming visit       │
│  ├─ Phrase   → gallop + position check (stream)   │
│  ├─ Boolean  → two-phase streaming conjunction    │
│  └─ Count    → u32 gallop / bitset union          │
└─────────────────────────────────────────────────┘
```

## Conclusion

We went from **13x slower** to **3.5x faster** than Lucene-BP through four focused optimizations, each targeting a specific bottleneck:

| Part | Bottleneck | Fix | Key Insight |
|------|-----------|-----|------------|
| 1 | Phrase query | Rarest-first + galloping + streaming | Lead with the smallest set |
| 2 | COUNT query | O(1) lookup + u32 SoA + bitset | Don't score what you don't need |
| 3 | Intersection | Two-phase streaming conjunction | Don't materialize what you can stream |
| 4 | (This post) | — | Simple algorithms + cache-friendly data > complex algorithms |

No exotic data structures. No complex algorithms. Just careful attention to **what data gets touched**, **in what order**, and **how much of it**.

---

*Read the full series:*
- *Part 1: [Phrase Query — From 4ms to 386μs](/posts/2026/search-query-optimization-part1-phrase-query/)*
- *Part 2: [COUNT Query — The Art of Not Scoring](/posts/2026/search-query-optimization-part2-count-query/)*
- *Part 3: [Intersection Query — Two-Phase Streaming Conjunction](/posts/2026/search-query-optimization-part3-intersection-query/)*
