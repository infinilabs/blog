---
title: "Shrinking a Search Engine to Fit in Your Browser — Part 5: The Last 60 KB"
meta_title: "WASM Size Optimization Series Part 5"
description: "After hitting 245 KB in Part 4, we kept digging. A nightly toolchain, a smaller allocator, a feature gate on legacy code, an unstable sort sweep, and finally dropping std entirely from the nano build — Pizza Engine's nano tier is now 185 KB gzipped, running as pure no_std+alloc."
date: "2026-05-24T10:00:00+08:00"
categories: ["Engineering"]
image: "/images/posts/2026/wasm-series/part5-the-last-60kb.jpeg"
author: "Medcl"
tags: ["WebAssembly", "Rust", "Search Engine", "Performance", "Pizza Engine"]
lang: "en"
category: "Engineering"
subcategory: "Deep Dive"
draft: false
---

In [Part 4](/posts/2026/05-13-wasm-series-4-optional-geo-vector/) we hit the original target: a full inverted-index search engine in **245 KB gzipped**. That should have been the end of the series.

It wasn't.

Once you can see the binary, you can't stop looking at it. A profile pass with `wasm-tools` showed obvious fat still in there: a 9 KB monomorphization of `core::cell::OnceCell::try_init`. An 8.7 KB family of `spin::Once` poll loops. A 52 KB `core::slice::sort::stable::driftsort` instantiation. Half of std's `panic` machinery, even though we'd already enabled `panic = "abort"`.

This post is the story of squeezing out another **60 KB gzipped** — from 245 KB down to **185 KB** — without losing a single feature. The final build is pure `no_std + alloc`: no standard library linked into our crate at all.

## Where We Started

After Part 4:

- 245 KB gzipped
- `release-wasm` profile (`opt-level = "z"`, `lto = true`, `codegen-units = 1`, `strip = true`)
- `wasm_nano` feature set: text search only, no JSON, no regex, no geo, no vector
- Stable Rust toolchain

The "easy" levers were already pulled. What followed was a measure → identify → eliminate → re-measure loop, with each round shaving anywhere from 100 bytes to 7 KB. No single trick was a hero. The result is cumulative.

## Tool 1: Profile What's Actually There

`twiggy` was the obvious first reach. It promptly died on the first `bulk-memory` opcode in our binary (`Unknown 0xfc opcode`). That tool hasn't been updated for the newer wasm proposals our toolchain emits.

So a 30-line Python script ([`/tmp/wasm_top.py`]) reading the wasm code section directly — parsing LEB-encoded function sizes and the name-section subsection — became the workhorse:

```bash
Total code: 380842 bytes across 2562 funcs
    Size       %  Name
    9038    2.37  serde_wasm_bindgen::de::Deserializer::deserialize_map
    8767    2.30  core::cell::once::OnceCell::try_init (×8 monos)
    6530    1.71  MmapFrozenSegment::open_v11_backing
    ...
```

This single output drove every subsequent decision. **You cannot optimize a binary you cannot see.**

To get readable names, build twice — once for shipping (stripped), once for profiling:

```bash
# Profile build: keep symbols
cargo +nightly rustc --profile release-wasm \
  --target wasm32-unknown-unknown \
  -Z build-std=std,panic_abort \
  --no-default-features --features wasm_nano \
  -- -C strip=none --emit=link

wasm-bindgen target/.../pizza_engine.wasm \
  --out-dir /tmp/wbg2 --target web --no-typescript --keep-debug
```

Then `wasm-tools print` is the only tool needed to enumerate functions by demangled name.

## Tool 2: Nightly + `build-std`

The single largest win in this round. The stock `std` ships with code paths we will never execute on `wasm32-unknown-unknown`: thread parking, `Once` state machines, panic unwinding, location-tracking diagnostics. With nightly's `build-std` we recompile `std` *from source* with our flags applied:

```bash
RUSTFLAGS="-Z location-detail=none \
           -Z unstable-options \
           -C panic=immediate-abort" \
cargo +nightly-2025-10-09 build --profile release-wasm \
  --target wasm32-unknown-unknown \
  -Z build-std=std,panic_abort \
  --no-default-features --features wasm_nano
```

What each flag does:

- **`-Z build-std=std,panic_abort`** — recompile std with our profile. Without this, you ship Rust's pre-built `std` which was compiled with neither `panic = abort` nor LTO across crate boundaries.
- **`-C panic=immediate-abort`** — `panic = "abort"` only stops unwinding. Panic *messages* and `core::panicking::panic_fmt` are still in the binary. `immediate-abort` replaces every panic with a bare `unreachable` opcode. Saves on the order of 20 KB of format machinery.
- **`-Z location-detail=none`** — strips `&'static str` file/line/column metadata from every panic site. Each removed `Location` is ~30 bytes of read-only data.
- **`--profile release-wasm`** specifies `opt-level = "z"`, `lto = true`, `codegen-units = 1`, `strip = true`, `panic = "abort"`.

**Result: 281 KB → 197 KB gzipped.** Single largest delta of the entire round.

The cost: you need a pinned nightly (`rust-toolchain.toml` to the rescue), `rust-src` component installed, and ~2× longer build times because std is rebuilt every clean cycle.

## Tool 3: A Smaller Allocator

`std`'s default allocator on `wasm32-unknown-unknown` is `dlmalloc`, weighing in around 10 KB. We don't need its thread safety — wasm is single-threaded. `lol_alloc` is a tiny bump-style free-list allocator written explicitly for this case:

```toml
# Cargo.toml
[target.'cfg(target_arch = "wasm32")'.dependencies]
lol_alloc = "0.4"
```

```rust
// src/lib.rs
#[cfg(target_arch = "wasm32")]
#[global_allocator]
static ALLOC: lol_alloc::AssumeSingleThreaded<lol_alloc::FreeListAllocator> =
    unsafe { lol_alloc::AssumeSingleThreaded::new(lol_alloc::FreeListAllocator::new()) };
```

`AssumeSingleThreaded` is the explicit promise that no other thread will ever call `alloc`. On wasm32 without the `atomics` proposal, that promise is enforced by the platform — there are no other threads. **−2.5 KB gzipped.**

## Tool 4: Feature-Gating Legacy Readers

Pizza Engine maintains a v10 segment reader for backward compatibility with older indexes. Nano consumers building fresh indexes in the browser will never encounter a v10 segment. So:

```toml
# Cargo.toml
fire_v10_compat = []

# included in: default, all, wasm_mini, wasm_ultra
# NOT included in: wasm_nano
```

```rust
// src/index/immutable/inverted/mmap/mod.rs
#[cfg(feature = "fire_v10_compat")]
pub(crate) fn open_v10_backing(...) -> Result<...> { /* 2 KB */ }

#[cfg(feature = "fire_v10_compat")]
fn parse_v10_field_metadata(...) -> Result<...> { /* 1 KB */ }

// ... 3 more cfg gates ...
```

LTO does most of this work for us when there are no callers, but explicit gates keep the surface area visible and protect against accidental cross-version coupling in future code. **−1.5 KB gzipped.**

## Tool 5: The Sort Sweep

This one surprised me. Our profile showed `core::slice::sort::stable::driftsort` taking **52 KB of raw wasm** — 12.86% of the entire binary. `driftsort` is Rust's modern stable sort: adaptive, merge-sort based, brilliant for general purpose, but it ships a *lot* of generated code per monomorphization (different key types → different specializations).

We don't need stable sort anywhere. Sort ties in the engine are either:
- already unique (doc IDs, field offsets), or
- broken by a secondary key, or
- semantically irrelevant (tie-breaks during top-K scoring).

So:

```bash
find src/ -name '*.rs' \
  | xargs sed -i '' \
      -e 's/\.sort_by_key(/.sort_unstable_by_key(/g' \
      -e 's/\.sort_by(/.sort_unstable_by(/g'
```

Unstable sort is `ipnsort` — pattern-defeating quicksort with introspection. Smaller code, comparable or *faster* runtime in our workload. Drift sort dropped from 52 KB to 26 KB raw.

**−5,977 bytes gzipped / −23,510 bytes raw.** Largest single edit-win of the entire round.

## Tool 6: One Stubborn `OnceLock`

By the late innings, the profile kept showing a fat `spin::once::Once::poll` loop and a fat `core::cell::OnceCell::try_init` — adding up to nearly 18 KB raw between them. We had a `OnceCellSync` shim in `src/util/once_cell.rs` mapping to `std::sync::OnceLock` on native, `spin::Once` on wasm. That was the wrong choice for nano.

First attempt: swap `spin::Once` to `core::cell::OnceCell` (single-threaded wasm doesn't need locking) with an `unsafe impl Sync` (safe because wasm has no threads):

```rust
#[cfg(target_arch = "wasm32")]
mod wasm_impl {
    use core::cell::OnceCell;

    pub struct OnceCellSync<T> { inner: OnceCell<T> }

    // Safe: wasm32 without atomics has no threads.
    unsafe impl<T: Send + Sync> Sync for OnceCellSync<T> {}

    impl<T> OnceCellSync<T> {
        pub const fn new() -> Self { Self { inner: OnceCell::new() } }
        pub fn get(&self) -> Option<&T> { self.inner.get() }
        pub fn set(&self, v: T) -> Result<(), T> { self.inner.set(v) }
        pub fn get_or_init<F: FnOnce() -> T>(&self, f: F) -> &T {
            self.inner.get_or_init(f)
        }
    }
}
```

That eliminated `spin::Once` (−165 bytes gzipped). Almost a no-op? Yes — because `OnceCell::try_init` then ballooned to take its place. The closures inside `get_or_init` get monomorphized per call site. Eight call sites = eight copies of init machinery. The bytes moved buckets, they didn't disappear.

Then I discovered the *real* spin user: `src/store/column/store.rs` had a parallel-track `Vec<spin::Once<DecodedColumn>>` for the wasm path, completely bypassing `OnceCellSync`. A leftover from an earlier optimization attempt. Removing the wasm-specific arm and routing everything through `OnceCellSync` finally killed the last `spin::*` symbols.

Net for this whole journey: **−2 KB gzipped, and zero `spin::` symbols in the binary.** The bigger win was conceptual — *every* `OnceLock`-style API in the codebase now goes through one shim, so future swaps are one-file changes.

## Tool 7: Dropping `std` Entirely — Pure `no_std + alloc`

The final architectural move. Our crate root already declared `#![no_std]` with conditional `extern crate std` behind `#[cfg(feature = "std")]`. The wasm module already imported from `alloc::` (Vec, String, Arc). But the `wasm` feature *forced* `std` on:

```toml
# Before: wasm pulls std unconditionally
wasm=["segment_reader", "std", "dep:wasm-bindgen", ...]
```

On inspection, nothing in the nano code path actually needs `std`. The standard library was there by inertia — inherited from when the wasm feature was first written. The column store had both `#[cfg(feature = "std")]` (lazy decode via OnceCellSync) and `#[cfg(not(feature = "std"))]` (eager decode) paths already implemented. All collection types come from `hashbrown` or `alloc`. Synchronization uses `spin::RwLock` or our `OnceCellSync` shim.

The fix was three lines of feature config:

```toml
# After: wasm is no_std+alloc; mini adds std
wasm=["segment_reader", "dep:wasm-bindgen", "dep:js-sys", "dep:web-sys", "dep:serde-wasm-bindgen"]
wasm_mini=["wasm_nano", "std", "wasm_fetch", ...]  # std added here
```

Plus replacing three leftover `std::mem::size_of` / `std::slice::from_raw_parts` calls with their `core::` equivalents in the BKD tree module. That's it.

The result: nano uses the `#[cfg(not(feature = "std"))]` column decode path (eager, no `OnceCell` machinery) and eliminates all `OnceLock`/`OnceCell::try_init` monomorphizations from the binary entirely.

**−1,238 bytes gzipped.** Small delta, huge architectural win — nano is now genuinely portable to any wasm runtime with a heap, not just browsers.

## Tool 8: Things That Looked Promising But Weren't

Two attempts cost more than they saved:

### Replacing `serde-wasm-bindgen` with `serde_json::to_string`

The thinking: `serde-wasm-bindgen` brings two heavy code paths — a JS-value walker for deserialization and a JS-value builder for serialization. `serde_json::to_string` should compress nicely under `opt-level = "z"` because it's pure byte writing.

The reality: serde_json's `CompactFormatter` monomorphizes per type, and our `SearchResult` is a rich enum tree. Net change: **+7 KB gzipped.** Reverted in full.

Lesson: **don't trust intuition about binary size. Measure every swap.**

### Inlining the BoolPlan expansion

A 1.2 KB function (`expand_wildcards`) seemed like a candidate for hand-rolling. After two hours of careful rewriting it came out 200 bytes *larger* because the LLVM optimizer was already doing a better job than my hand-rolled version. Reverted.

## The Feature Matrix

Pizza Engine ships four WASM tiers. Each adds capabilities on top of the previous:

| | **nano** | **micro** | **mini** | **ultra** |
|---|---|---|---|---|
| **Runtime** | `no_std + alloc` | `std` | `std` | `std` |
| **Core search** | | | | |
| Segment reader (mmap/bytes) | ✅ | ✅ | ✅ | ✅ |
| Text search (term/match/prefix/wildcard/bool) | ✅ | ✅ | ✅ | ✅ |
| BM25 scoring | ✅ | ✅ | ✅ | ✅ |
| Column store / field retrieval | ✅ (eager) | ✅ (lazy) | ✅ (lazy) | ✅ (lazy) |
| Fuzzy search (Levenshtein) | ✅ | ✅ | ✅ | ✅ |
| **Query features** | | | | |
| Query string parser | ❌ | ✅ | ✅ | ✅ |
| Geo queries (bbox, distance) | ❌ | ✅ | ✅ | ✅ |
| Vector queries (KNN) | ❌ | ✅ | ✅ | ✅ |
| Graph traversal | ❌ | ✅ | ✅ | ✅ |
| JSON DSL queries | ❌ | ❌ | ✅ | ✅ |
| Regex queries | ❌ | ❌ | ❌ | ✅ |
| **Advanced** | | | | |
| Aggregations (terms, histogram, stats) | ❌ | ❌ | ❌ | ✅ |
| Scripting (Rhai) | ❌ | ❌ | ❌ | ✅ |
| Fetch API (remote segments) | ❌ | ❌ | ✅ | ✅ |
| Legacy v10 segment reader | ❌ | ✅ | ✅ | ✅ |

The key design insight: each tier adds one "cost band" of features. The `micro` tier adds all query types that are **cheap in code size** (geo, vector, graph, query parser — all under 60 KB combined). The expensive features — JSON parsing (357 KB!), regex (52 KB), and aggregations (115 KB with serde) — land in the higher tiers where binary size is less critical.

The feature composition in `Cargo.toml`:

```toml
wasm_nano  = ["wasm"]                       # no_std+alloc, text search only
wasm_micro = ["wasm_nano", "std", "wasm_panic_hook", "geo_queries",
              "vector_queries", "query_string_parser", "graph",
              "fire_v10_compat"]             # all query types, typed API
wasm_mini  = ["wasm_micro", "json", "wasm_fetch"]  # + JSON API, remote fetch
wasm_ultra = ["wasm_mini", "aggs", "regex_queries", "scripting"]  # full analytics
```

## Per-Feature Size Cost

Each feature measured in isolation on top of `wasm_nano` (no interaction effects):

| Feature | Gzip Delta | Why |
|---|---|---|
| `vector_queries` | ~0 | Marker only — code shared with text path |
| `std` | +1 KB | Sysroot overhead (no rayon on wasm) |
| `wasm_panic_hook` | +2 KB | `console_error_panic_hook` |
| `fire_v10_compat` | +4 KB | v10 on-disk format reader |
| `aggs` (without json) | +4 KB | Aggregation executor logic |
| `graph` | +7 KB | Adjacency index + traversal |
| `query_string_parser` | +16 KB | Hand-written PEG-style parser |
| `wasm_fetch` | +20 KB | wasm-bindgen-futures + Request/Response |
| `geo_queries` | +33 KB | Haversine + bounding-box |
| `regex_queries` | +52 KB | regex-automata DFA engine |
| `aggs` (WITH json) | +115 KB | Serde Deserialize for 40-variant enum |
| **`json`** | **+357 KB** | **serde_json + all query type deserializers** |

The critical insight: `json` alone accounts for **48% of the mini binary**. The `aggs` feature costs only +4 KB in isolation but +115 KB when combined with `json` — because `serde` generates a Deserialize impl for each of the 40 aggregation variants.

## Size by Tier

All measurements: `release-wasm` profile, nightly `build-std`, full pipeline (`wasm-bindgen` → `wasm-opt -Oz --converge`), `gzip -9` / `brotli -q 11`.

| Tier | Raw WASM | Gzip | Brotli | Use case |
|------|----------|------|--------|----------|
| **nano** | 407 KB | **187 KB** | **152 KB** | Edge workers, Service Workers |
| **micro** | 555 KB | **244 KB** | **197 KB** | Typed SDK, no JSON overhead |
| **mini** | 1,698 KB | **605 KB** | **458 KB** | Standard web apps (JSON API) |
| **ultra** | 2,379 KB | **792 KB** | **579 KB** | Full analytics in browser |

## The Progression

```bash
Total code: 377,750 bytes across 2,530 functions (final nano)

                                 gzip       raw
Baseline (Part 4):               245,000    ~768,000
+ build-std + immediate-abort:   197,000    ~449,000   (−48 KB gzip)
+ Drop panic_hook:               196,935    ~444,000   (−65 B)
+ lol_alloc allocator:           ~194,000   ~437,000   (−2.5 KB)
+ fire_v10_compat gate:          192,453    ~432,808   (−1.5 KB)
+ sort_unstable sweep:           186,476    ~409,298   (−6 KB)
+ OnceCellSync unification:      186,252    ~409,140   (−0.2 KB)
+ Drop std from nano (no_std):   185,014    ~406,898   (−1.2 KB)

Final: 185,014 gzipped / 151,862 brotli / 406,898 raw
```

## What's Actually In Those 185 KB

| Component | Raw bytes | % |
|---|---|---|
| Inverted index codecs + query execution | ~80,000 | 21% |
| `core::slice::sort::ipnsort` (unstable sort) | ~26,000 | 7% |
| `hashbrown::RawTable::reserve_rehash` (×6 monos) | ~15,500 | 4% |
| `MmapFrozenSegment::open_v11_backing` | ~12,000 | 3% |
| `serde_wasm_bindgen` deserializer | ~11,000 | 3% |
| `fst-no-std` (FST tries + Levenshtein DFA) | ~10,000 | 3% |
| `core::num::flt2dec` (float Display) | ~8,900 | 2% |
| `ColumnStore::new` (eager decode, no_std path) | ~8,600 | 2% |
| Everything else | ~206,000 | 55% |

## Practical Notes

A few lessons that generalize:

1. **`build-std` is the biggest lever you're probably not pulling.** If you ship to a constrained target — wasm, embedded, AVR — recompiling `std` with your panic and location-detail flags is worth more than every other micro-optimization combined.

2. **Profile before you cut.** I spent an evening hand-rolling a faster wildcard expander. It made the binary bigger. The compiler is smarter than you on cold paths.

3. **Stable sort is expensive.** Rust's `driftsort` is excellent at runtime but heavy at compile time. If your sort keys are unique or your ties don't matter, `sort_unstable_*` is a free win every time.

4. **Closures monomorphize.** Every call site of `something.get_or_init(|| expensive_closure())` produces a fresh instantiation. If you have eight call sites and they're all heavy, that's eight copies in your binary. Sometimes a `&dyn Fn() -> T` parameter pays for itself.

5. **Removing a dependency only helps if you remove *all* its users.** Replacing `spin::Once` in our shim did almost nothing — until we found the other direct `spin::Once` user we'd forgotten about.

6. **`no_std` isn't just for embedded.** Even with `build-std=std,panic_abort` (because transitive deps need std in the sysroot), disabling the `std` feature in your *own* crate eliminates std-gated code paths — OnceLock machinery, thread-local storage, io::Error conversions. The delta is modest (~1 KB gzip) but the portability gain is real.

## Where Next?

We're stopping the raw byte-counting here. But the final act was **tier redesign**: instead of the original 3-tier system (nano/mini/ultra), we landed on a 4-tier architecture that's data-driven by the per-feature measurements above.

The new `micro` tier is the secret weapon: you get geo queries, vector search, graph traversal, and query string parsing for only **+57 KB gzip** over nano. The expensive jump is `nano → mini` (+418 KB), which is almost entirely `serde_json` pulling in Deserialize impls for every query type. If your JS layer can construct typed query objects directly — via a thin SDK that maps to Rust structs through `serde-wasm-bindgen` — you skip the JSON tax entirely and stay in the 244 KB band.

```bash
nano (187 KB)  ──+57 KB──▶  micro (244 KB)  ──+361 KB──▶  mini (605 KB)  ──+187 KB──▶  ultra (792 KB)
     │                            │                              │                            │
  text search              + geo, vector,                   + JSON API,                 + aggs, regex,
  no_std+alloc              graph, QSP                      remote fetch                 scripting
```

The remaining optimizations from earlier (schema-aware deserializer, hashbrown monomorphization reduction, float display → ryu) still apply to each tier — but the biggest wins now are **architectural**: choosing the right tier for your deployment, rather than shaving bytes from code you actually need.

---

*Pizza Engine is an embedded search engine written in Rust. The WASM tiers deliver instant in-browser search without a server — from the 187 KB `nano` (pure `no_std + alloc`, portable to any runtime with a heap) to the 792 KB `ultra` (full aggregations, regex, vector search, geo queries, graph traversal, and scripting). Source on [GitHub](https://github.com/pizza-rs/engine).*
