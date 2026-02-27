---
title: "Search Query Optimization Part 3: Intersection Query — Two-Phase Streaming Conjunction"
meta_title: "Search Query Optimization Part 3: Intersection Query — Two-Phase Streaming Conjunction"
description: "How we eliminated the stopword intersection bottleneck in Pizza Engine — making queries like '+the +incredibles' 12x faster than Lucene-BP, without implementing Block-max WAND."
date: "2026-02-27T10:00:00+08:00"
categories: ["Engineering", "Performance"]
image: "/images/posts/2026/search-query-optimization-part3.png"
author: "Medcl"
tags: ["Pizza Engine","FIRE", "Search", "Performance", "Rust", "Intersection", "Streaming"]
lang: "en"
category: "Engineering"
subcategory: "Performance"
draft: false
---

*This is Part 3 of a series on optimizing Pizza Engine's search queries. After tackling phrase queries (Part 1) and COUNT queries (Part 2), we had one remaining bottleneck: TOP_K intersection queries with stopwords. This post shows how a streaming two-phase design eliminated the problem — without implementing Block-max WAND.*

## The Remaining Bottleneck

After Parts 1 and 2, our overall numbers looked good: TOP_10 at 0.38x Lucene-BP (2.6x faster), TOP_100 at 0.46x (2.2x faster). But the bottleneck analysis revealed extreme outliers:

| Query | Pizza (μs) | Lucene-BP (μs) | Ratio |
|-------|--------:|--------:|------:|
| `+the +incredibles` | 728 | 54 | **13.5x slower** |
| `+american +academy +of +child ...` | 4,312 | 247 | **17.5x slower** |
| `+the +oregonian +newspaper` | 1,134 | 100 | **11.3x slower** |
| `+the +daily +breeze` | 1,046 | 247 | **4.2x slower** |

All the worst queries had a common pattern: **intersection with at least one very common term** (stopwords like "the", "of", "and", or semi-stopwords like "american").

## Why Lucene Was Faster: Block-max WAND

Lucene uses an algorithm called **Block-max WAND** (Weak-AND) for TOP_K queries. The idea:

1. Posting lists are divided into **blocks** (e.g., 128 docs per block)
2. Each block stores the **maximum score** any document in that block can achieve
3. During query evaluation, if a block's max-score can't beat the current K-th best score in the heap, the **entire block is skipped**

Combined with **BP (Bipartite graph Partitioning) doc-ID reordering** — which clusters similar documents together so they share blocks — this makes stopword-heavy queries fast. If you search for `+the +incredibles`, Lucene can skip >99% of "the"'s posting list because most blocks can't score high enough for "the" alone to contribute.

The question was: do we need to implement Block-max WAND?

## Diagnosis: The Real Problem

Let's trace what happened for `+the +incredibles` in our code:

```text
search_boolean("+the +incredibles")
  ├─ search("+the")         → query_term("the")    → Vec<Hit> with 4,012,345 entries! (30MB allocation)
  ├─ search("+incredibles") → query_term("incredibles") → Vec<Hit> with 247 entries
  └─ merge_and()            → intersect → ~247 results
```

The problem wasn't the intersection algorithm. The problem was **Step 1**: we materialized the entire posting list of "the" into a `Vec<Hit>` — 4 million `Hit` structs (8 bytes each, 30+ MB) — before the intersection even started.

And this happened inside `search_boolean` which operated on `Vec<Hit>`:

```rust
fn search_boolean(&self, bq: &BooleanQuery) -> Vec<Hit> {
    let must_hits: Vec<Vec<Hit>> = bq.must.iter()
        .map(|q| self.search(q))  // ← materializes EVERYTHING
        .collect();
    self.merge_and(&must_hits)    // ← then intersects
}
```

The caller (`search_visit`) would then iterate the result and feed it to a `TopKCollector`:

```rust
// In search_visit — the fallback path for Boolean queries:
let hits = self.search(query);  // ← all results materialized
for hit in hits {
    if !visitor(hit) { return; }  // TopK early termination
}
```

Even though the visitor could terminate early, the entire `search()` call had to complete first. Early termination was useless.

## The Design: Two-Phase Streaming Conjunction

Instead of `search()` → `Vec<Hit>` → `visit`, we needed to **stream results directly** from posting list references. The design:

**Phase 1 — Existence check (u32 galloping):** For each candidate doc in the rarest term, gallop through other terms' `doc_ids: Vec<u32>` arrays (4 bytes/entry, cache-efficient) to verify the document exists in all terms.

**Phase 2 — Score lookup (PostingEntry):** Only for documents that pass Phase 1, look up `entries[idx].score` to get the pre-computed BM25 score.

```text
Before (materialize-all):
  "the"          → 4M Hit structs → 30MB alloc
  "incredibles"  → 247 Hit structs
  merge_and()    → ~247 results
  visit()        → feed to TopK

After (two-phase streaming):
  Resolve postings refs: "incredibles" (247 docs), "the" (4M docs)
  For each of 247 docs in "incredibles":
    Phase 1: gallop "the".doc_ids for existence → O(log d) on 4B/entry
    Phase 2: score = incredibles.entries[i].score + the.entries[j].score
    visitor(Hit) → TopK can stop early!
```

Zero allocation. 247 iterations instead of 4 million. Cache-efficient. Early termination works.

## Implementation

The key method is `search_boolean_and_visit`:

```rust
fn search_boolean_and_visit<F>(
    &self, ctx, schema, bq, default_field,
    visitor: &mut F,
) where F: FnMut(Hit) -> bool {

    // Step 1: Resolve all must clauses to TermPostings references
    let mut term_postings: Vec<&TermPostings> = bq.must.iter()
        .filter_map(|q| self.resolve_term_postings(q))
        .collect();

    // Step 2: Sort by doc_frequency — rarest first
    term_postings.sort_unstable_by_key(|tp| tp.doc_frequency);

    // Step 3: Dedup identical posting lists (repeated terms)
    term_postings.dedup_by(|a, b| {
        core::ptr::eq(a.doc_ids.as_ptr(), b.doc_ids.as_ptr())
    });

    let driver = term_postings[0];  // rarest term
    let others = &term_postings[1..];
    let mut cursors = vec![0usize; others.len()];

    // Step 4: Stream through the rarest term
    for entry in &driver.entries {
        let doc_id = entry.doc_id;
        let mut total_score = entry.score;

        // Phase 1+2: gallop + score in one pass
        for (j, other) in others.iter().enumerate() {
            match gallop_search_u32(&other.doc_ids, doc_id, cursors[j]) {
                Some(idx) => {
                    total_score += other.entries[idx].score;
                    cursors[j] = idx + 1;
                }
                None => {
                    if cursors[j] >= other.doc_ids.len() {
                        return;  // other list exhausted — done
                    }
                    continue 'outer;  // doc not in this term
                }
            }
        }

        // Feed to TopK collector — can stop early!
        if !visitor(Hit { doc_id, score: total_score }) {
            return;
        }
    }
}
```

This is wired into `search_visit` for Boolean queries where all must clauses are simple terms:

```rust
Query::Boolean(bq) => {
    if all_must_clauses_are_simple_terms(bq) {
        self.search_boolean_and_visit(ctx, schema, bq, default_field, visitor);
    } else {
        // Fallback: materialize then visit
        let hits = self.search(query);
        for hit in hits { if !visitor(hit) { return; } }
    }
}
```

## Why Not Block-max WAND?

Block-max WAND is a powerful technique, but it requires:

1. **Block-based posting list format** — restructure the entire index format
2. **Per-block max-score metadata** — additional storage and computation at build time
3. **Dynamic threshold tracking** — the algorithm maintains a running threshold from the top-K heap during traversal
4. **Scored traversal** — WAND operates on scores, so every document must be partially scored even if skipped

Our two-phase approach sidesteps all of this:

| Aspect | Block-max WAND | Two-Phase Streaming |
|--------|---------------|-------------------|
| Index format change | Required (block structure) | None |
| Build-time overhead | Per-block max-score computation | None (u32 array already exists) |
| Algorithm complexity | High (threshold tracking, pivot selection) | Low (galloping + cursor) |
| Works for COUNT too | No (score-based) | u32 arrays shared with COUNT |
| Early termination | Via threshold pruning | Via visitor pattern |

The key insight: **for intersection queries, the rarest-term-first approach already eliminates most work**. If "incredibles" has 247 postings, we only do 247 iterations regardless of how many documents "the" has. Block-max WAND would also iterate ~247 relevant blocks, but with much more per-block overhead.

Where Block-max WAND wins is **disjunctive (OR) TOP_K** queries where you need to find the highest-scoring documents across multiple lists without visiting all of them. That's a different bottleneck we may address later.

## Results

| Category | Before (P/BP) | After (P/BP) | Improvement |
|----------|------:|------:|-----:|
| TOP_10 intersection | 0.27x | **0.08x** | 3.4x faster |
| TOP_10 OVERALL | 0.38x | **0.29x** | 1.3x faster |
| TOP_100 intersection | 0.52x | **0.21x** | 2.5x faster |
| TOP_100 OVERALL | 0.46x | **0.38x** | 1.2x faster |

The outliers were obliterated:

| Query | Before (P/BP) | After (P/BP) |
|-------|------:|------:|
| `+the +incredibles` | 13.5x slower | **Gone from top-15** |
| `+american +academy +of +child ...` | 17.5x slower | **Gone from top-15** |
| `+the +oregonian +newspaper` | 11.3x slower | **Gone from top-15** |

The worst TOP_10 P/BP ratio dropped from **17.5x** to **6.3x** (now union queries, not intersection). Median TOP_10 intersection went from 146μs to **44μs** — meaning Pizza is now **12.5x faster** than Lucene-BP on intersection queries.

## Design Principles

Three design principles emerged from this optimization:

1. **Don't materialize what you can stream.** The `Vec<Hit>` was the bottleneck, not the intersection algorithm. By keeping posting list references and streaming results, we eliminated the allocation entirely.

2. **Let the data structure serve multiple purposes.** The `doc_ids: Vec<u32>` array was originally built for COUNT queries (Part 2). Here it doubles as the Phase 1 existence-check structure for TOP_K queries. One data structure, two use cases.

3. **Simple beats complex when the constant factor is small enough.** Galloping + rarest-first + streaming is algorithmically simpler than Block-max WAND, but it has a tiny constant factor because it operates on contiguous, cache-friendly arrays with no dynamic threshold management.

---

*Part 2: [COUNT Query — The Art of Not Scoring](/posts/2026/search-query-optimization-part2-count-query/)*

*Part 4: [Beating Lucene — Final Scorecard](/posts/2026/search-query-optimization-part4-final-scorecard/)*
