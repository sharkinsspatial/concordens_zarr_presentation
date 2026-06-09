# Zarr v3 Presentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a 20-minute Slidev presentation covering Zarr v3 sharding, codec pipeline, zarrs-python, rectilinear chunking, Icechunk, and v2→v3 migration for Zarr v2 power users at Caur Tech.

**Architecture:** Single Slidev markdown project with four Excalidraw diagrams embedded via `slidev-addon-excalidraw`. Slides are problem-driven — each section frames a pain point, then introduces the solution.

**Tech Stack:** Slidev, `@slidev/theme-seriph`, `slidev-addon-excalidraw`, Excalidraw JSON format

**Spec:** `docs/superpowers/specs/2026-06-09-zarr-v3-presentation-design.md`

---

## File Structure

```
slides.md                          — All slide content (Slidev single-file format)
package.json                       — Dependencies (created by npm init slidev)
public/
  diagrams/
    sharding-concept.excalidraw    — Sharding inner/outer chunks diagram
    codec-pipeline.excalidraw      — Codec pipeline flow diagram
    rectilinear-chunking.excalidraw — Uniform vs. rectilinear comparison
    icechunk-architecture.excalidraw — Icechunk writers/branches/snapshots
```

---

### Task 1: Scaffold Slidev Project

**Files:**
- Create: `package.json` (via npm init)
- Create: `slides.md` (initial scaffold, will be replaced)

- [ ] **Step 1: Initialize the Slidev project**

Run:
```bash
cd /Users/seanharkins/projects/zarr_presentation
npm init slidev@latest -- --name zarr-v3-presentation --entry slides.md
```

Follow prompts — accept defaults. This creates `package.json`, `slides.md`, and supporting directories.

- [ ] **Step 2: Install the Excalidraw addon**

Run:
```bash
npm install slidev-addon-excalidraw
```

- [ ] **Step 3: Create the diagrams directory**

Run:
```bash
mkdir -p public/diagrams
```

- [ ] **Step 4: Set up the slide frontmatter**

Replace the content of `slides.md` with the deck-level frontmatter and title slide:

```md
---
theme: seriph
title: "Zarr v3 in Practice"
info: |
  Sharding, Codecs, and Icechunk for High-Throughput Instrument Data
addons:
  - slidev-addon-excalidraw
highlighter: shiki
transition: fade
---

# Zarr v3 in Practice

## Sharding, Codecs, and Icechunk for High-Throughput Instrument Data

<div class="abs-br m-6 text-sm opacity-50">
Sean Harkins · June 2026
</div>
```

- [ ] **Step 5: Verify the dev server starts**

Run:
```bash
npx slidev --open false
```

Expected: Server starts on `http://localhost:3030` with no errors. Stop it with Ctrl+C after confirming.

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json slides.md public/
git commit -m "feat: scaffold Slidev project with seriph theme and excalidraw addon"
```

---

### Task 2: Create Excalidraw Diagrams

**Files:**
- Create: `public/diagrams/sharding-concept.excalidraw`
- Create: `public/diagrams/codec-pipeline.excalidraw`
- Create: `public/diagrams/rectilinear-chunking.excalidraw`
- Create: `public/diagrams/icechunk-architecture.excalidraw`

Each diagram uses the Excalidraw JSON format. Style: clean, schematic — boxes, arrows, labels. No decorative elements. Use a consistent color palette across all four diagrams.

- [ ] **Step 1: Create the sharding concept diagram**

Write `public/diagrams/sharding-concept.excalidraw`:

This diagram shows:
- Left side: a 4x4 grid of small squares labeled "Logical Chunks" (each labeled c0,0 through c3,3)
- Arrow pointing right labeled "ShardingCodec"
- Right side: a single larger rectangle labeled "Shard (1 storage object)" containing the same 4x4 grid of inner chunks packed together, plus a small strip at the bottom labeled "Shard Index" with byte offset entries
- Caption below: "256 inner chunks → 1 S3 object (instead of 256)"

Use these colors consistently across all diagrams:
- Light blue (#a5d8ff) for data/chunks
- Light orange (#ffd8a8) for metadata/index
- Light green (#b2f2bb) for output/storage
- Dark gray (#495057) for labels and arrows

The Excalidraw JSON format is an object with `type: "excalidraw"`, `version: 2`, `source: "manual"`, and an `elements` array. Each element has properties like `type` ("rectangle", "text", "arrow"), `x`, `y`, `width`, `height`, `strokeColor`, `backgroundColor`, `fillStyle`, and `text` (for text elements). Create the full JSON with all elements positioned to produce the described diagram.

- [ ] **Step 2: Create the codec pipeline diagram**

Write `public/diagrams/codec-pipeline.excalidraw`:

This diagram shows a left-to-right flow:
- Box 1: "Raw Array" (light blue background)
- Arrow → Box 2: "Array → Array" with subtitle "TransposeCodec, DeltaCodec" (light blue)
- Arrow → Box 3: "Array → Bytes" with subtitle "BytesCodec (endianness)" (light orange)
- Arrow → Box 4: "Bytes → Bytes" with subtitle "BloscCodec, ZstdCodec, Crc32cCodec" (light orange)
- Arrow → Box 5: "Stored Bytes" (light green)

Each arrow labeled with the data type flowing between stages ("ndarray", "ndarray", "bytes", "bytes").

- [ ] **Step 3: Create the rectilinear chunking diagram**

Write `public/diagrams/rectilinear-chunking.excalidraw`:

This diagram shows two side-by-side 2D grids:
- Left grid labeled "Uniform Chunks": a 4x4 grid where all cells are the same size. Dimension labels show equal spacing.
- Right grid labeled "Rectilinear Chunks": a 4x4 grid where columns have varying widths (e.g., narrow, wide, medium, narrow) representing different chunk sizes along one axis. Dimension labels show `[64, 128, 96, 64]` along the variable axis and `[256, 256, 256, 256]` along the uniform axis.
- Caption: "Variable-length acquisition windows → chunks that match the data"

- [ ] **Step 4: Create the Icechunk architecture diagram**

Write `public/diagrams/icechunk-architecture.excalidraw`:

This diagram shows:
- Top: two boxes labeled "Instrument A" and "Instrument B" (light blue), each with an arrow down labeled "write session"
- Middle: a rounded rectangle labeled "Icechunk" containing:
  - Two branch labels: "branch: main" and "branch: experiment-1"
  - A chain of circles/nodes representing snapshots (s1 → s2 → s3) with the branch pointer at the head
- Bottom: a large rectangle labeled "Object Storage (S3/GCS)" (light green) with icons for "manifests", "chunk refs", "snapshots"
- Side annotation: "No external DB required"

- [ ] **Step 5: Commit**

```bash
git add public/diagrams/
git commit -m "feat: add four Excalidraw diagrams for presentation"
```

---

### Task 3: Write Slides — Sections 1-2 (Title, Context, Sharding)

**Files:**
- Modify: `slides.md`

- [ ] **Step 1: Add the framing slide (Slide 2)**

Append to `slides.md` after the title slide:

```md
---
layout: center
---

# You Know Zarr v2

Today: what the **v3 ecosystem** solves for high-throughput, cloud-targeted instrument pipelines

<v-clicks>

- 🔹 Small-object overhead → **Sharding**
- 🔹 Codec rigidity & Python overhead → **v3 Codec Pipeline + zarrs-python**
- 🔹 Chunking inflexibility → **Rectilinear Chunks**
- 🔹 Concurrent write safety → **Icechunk**

</v-clicks>
```

- [ ] **Step 2: Add the sharding problem slide (Slide 3)**

```md
---
layout: default
---

# The Small-Object Problem

Writing high-frequency instrument data to cloud storage with Zarr v2:

<v-clicks>

- Each logical chunk = **one storage object**
- 1M chunks = 1M S3 objects
- **S3 LIST** calls: $0.005 per 1,000 requests
- **GET latency**: each object = separate HTTP round-trip
- Metadata overhead grows linearly with object count

</v-clicks>

<!--
For a tomography array at high sample rates, you can easily hit millions of tiny objects.
The fundamental issue: v2 couples chunk granularity to storage object count.
-->
```

- [ ] **Step 3: Add the sharding concept slide with diagram (Slide 4)**

```md
---
layout: default
---

# Sharding: Decouple Chunks from Objects

<Excalidraw drawFilePath="/diagrams/sharding-concept.excalidraw" :darkMode="false" class="w-full h-80" />

- **Inner chunks**: fine-grained for partial reads
- **Shard (outer chunk)**: one storage object containing many inner chunks
- **Shard index**: byte offsets enable reading a single inner chunk without fetching the whole shard
```

- [ ] **Step 4: Add the sharding code slide (Slide 5)**

```md
---
layout: default
---

# Sharding in zarr-python 3.x

```python {all|3-8|4|5-8}
import zarr
from zarr.codecs import ShardingCodec, BloscCodec, BytesCodec

arr = zarr.open_array(
    "s3://bucket/tomography.zarr/data",
    shape=(10000, 4096, 4096),
    chunks=(100, 512, 512),         # shard (outer) shape
    codecs=[
        ShardingCodec(
            chunk_shape=(10, 32, 32),  # inner chunk shape
            codecs=[BytesCodec(), BloscCodec(cname="zstd", clevel=3)],
        )
    ],
    dtype="float32",
)
```

Each shard: `(100×512×512)` = **~100 MB** — one S3 object
Inner chunks: `(10×32×32)` — partial reads at fine granularity
```

- [ ] **Step 5: Add the sharding trade-offs slide (Slide 6)**

```md
---
layout: two-cols
---

# Sharding Trade-offs

**Benefits**

- Reduces object count by 100-1000x
- Lower cloud API costs
- Better sequential throughput
- Shard index enables partial reads

::right::

**Considerations**

- **Write amplification**: updating one inner chunk rewrites the shard
- Shard size sweet spot: **1-100 MB**
- Too large → wasted bandwidth on partial reads
- Too small → back to the small-object problem
- Best when write pattern is **append-heavy** (not random updates)

<!--
For tomography data that's written once and read many times, sharding is almost always a win.
Random single-chunk updates are the main case where sharding hurts.
-->
```

- [ ] **Step 6: Commit**

```bash
git add slides.md
git commit -m "feat: add title, context, and sharding slides (sections 1-2)"
```

---

### Task 4: Write Slides — Section 3 (Codec Pipeline & zarrs-python)

**Files:**
- Modify: `slides.md`

- [ ] **Step 1: Add the codec problem slide (Slide 7)**

Append to `slides.md`:

```md
---
layout: default
---

# v2 Codec Limitations

```python
# Zarr v2: flat, implicit ordering
zarr.open_array(
    ...,
    compressor=Blosc(cname="zstd", clevel=3),
    filters=[Delta(dtype="float32")],
)
```

<v-clicks>

- Filter vs. compressor distinction is **arbitrary**
- Ordering between filters is implicit
- No checksums in the pipeline
- All codec work happens in **Python** — GIL-bound

</v-clicks>
```

- [ ] **Step 2: Add the v3 codec pipeline slide with diagram (Slide 8)**

```md
---
layout: default
---

# v3: Typed, Ordered Codec Pipeline

<Excalidraw drawFilePath="/diagrams/codec-pipeline.excalidraw" :darkMode="false" class="w-full h-60" />

| Stage | Role | Examples |
|-------|------|----------|
| **Array → Array** | Transform array data | `TransposeCodec`, delta, scaling |
| **Array → Bytes** | Serialize to bytes | `BytesCodec` (explicit endianness) |
| **Bytes → Bytes** | Compress / checksum | `BloscCodec`, `ZstdCodec`, `Crc32cCodec` |

Codecs execute in **declared order** — no ambiguity.
```

- [ ] **Step 3: Add the codec pipeline code slide (Slide 9)**

```md
---
layout: default
---

# Codec Pipeline in Code

```python {all|3-8}
import zarr
from zarr.codecs import TransposeCodec, BytesCodec, BloscCodec, Crc32cCodec

arr = zarr.open_array(
    "data.zarr",
    shape=(4096, 4096),
    dtype="float32",
    codecs=[
        TransposeCodec(order=(1, 0)),       # array → array: column-major
        BytesCodec(endian="little"),         # array → bytes
        BloscCodec(cname="zstd", clevel=3), # bytes → bytes: compress
        Crc32cCodec(),                      # bytes → bytes: checksum
    ],
)
```

Explicit, composable, verifiable. Each stage has a clear contract.
```

- [ ] **Step 4: Add the zarrs-python slide (Slide 10)**

```md
---
layout: default
---

# zarrs-python: Rust-Powered Codecs

Drop-in replacement for zarr-python's codec pipeline, backed by the **zarrs** Rust crate via PyO3.

```python
pip install zarrs
```

```python {all|2}
import zarr
zarr.config.set({"codec_pipeline.path": "zarrs.ZarrsCodecPipeline"})

# All existing zarr code works unchanged — codecs run in Rust
arr = zarr.open_array("data.zarr", mode="r")
data = arr[:]
```

<v-clicks>

- Bypasses Python GIL for codec work
- Configurable concurrency: `chunk_concurrent_maximum`, `chunk_concurrent_minimum`
- Supports `validate_checksums` and `direct_io` (O_DIRECT) options

</v-clicks>
```

- [ ] **Step 5: Add the zarrs-python caveats slide (Slide 11)**

```md
---
layout: two-cols
---

# zarrs-python: Performance & Caveats

**Where it shines**

- Write-heavy workloads: biggest speedup
- Sharded data: parallelizes inner chunk codec work
- Large contiguous reads
- Configurable thread pool per operation

::right::

**Current limitations**

- **Filesystem stores only** — no S3/GCS/Azure yet
- Best for local write pipelines, post-acquisition processing
- v0.2.3 — API is still stabilizing
- Falls back to Python pipeline for unsupported stores

<!--
For Caur Tech: if instruments write to local disk first then sync to cloud,
zarrs-python can accelerate the local write path significantly.
-->
```

- [ ] **Step 6: Commit**

```bash
git add slides.md
git commit -m "feat: add codec pipeline and zarrs-python slides (section 3)"
```

---

### Task 5: Write Slides — Section 4 (Chunking, Access Patterns, Rectilinear Chunks)

**Files:**
- Modify: `slides.md`

- [ ] **Step 1: Add the chunking strategy slide (Slide 12)**

Append to `slides.md`:

```md
---
layout: default
---

# Chunking Strategy: Align to Access Patterns

```
Time axis ──────────────────────────►
Frequency  ┌────┬────┬────┬────┬────┐
axis       │    │    │    │    │    │  ← chunk along time: fast time slices
│          │    │    │    │    │    │
▼          ├────┼────┼────┼────┼────┤
           │    │    │    │    │    │
           └────┴────┴────┴────┴────┘
```

<v-clicks>

- **Time-series queries** → chunk along time axis (narrow, deep chunks)
- **Spectral/frequency queries** → chunk along frequency axis
- **Both?** → compromise shape, or use sharding with fine inner chunks
- Rule of thumb: optimize for the **80% query pattern**

</v-clicks>
```

- [ ] **Step 2: Add the object-store best practices slide (Slide 13)**

```md
---
layout: default
---

# Object-Store Best Practices

<v-clicks>

- **Chunk size sweet spot: 1–100 MB** per storage object
  - Below 1 MB: HTTP overhead dominates
  - Above 100 MB: wasted bandwidth on partial reads
- **Consolidated metadata**: avoid per-array metadata fetches
  - zarr-python 3.x: `zarr.consolidate_metadata(store)`
- **Flat hierarchies**: every group level = additional LIST call
- **Read coalescing**: merge nearby byte ranges into fewer requests
  - S3 supports multi-range GET (check client support)
- **Concurrent reads**: use async stores or thread pools for parallel chunk fetches

</v-clicks>
```

- [ ] **Step 3: Add the rectilinear chunking concept slide with diagram (Slide 14)**

```md
---
layout: default
---

# Rectilinear Chunking (ZEP 3)

<Excalidraw drawFilePath="/diagrams/rectilinear-chunking.excalidraw" :darkMode="false" class="w-full h-70" />

Per-dimension chunk size **lists** instead of uniform integers.
No more padding or artificial splitting for variable-length data.
```

- [ ] **Step 4: Add the rectilinear use cases slide (Slide 15)**

```md
---
layout: default
---

# Rectilinear Chunking: Use Cases

<v-clicks>

- **Variable-length acquisitions**: instruments produce runs of different durations
  - Chunks match actual data boundaries — no wasted padding
- **Virtualizing heterogeneous collections**: concatenating NetCDF/HDF5 files of different sizes along the time axis
  - Each file → one chunk along concat dimension
- **Run-length encoding** in metadata for efficiency when many chunks share the same size

</v-clicks>
```

- [ ] **Step 5: Add the rectilinear code slide (Slide 16)**

```md
---
layout: default
---

# Rectilinear Chunks in Code

```python {all|1-2|4-9}
import zarr
zarr.config.set({"array.rectilinear_chunks": True})  # experimental flag

arr = zarr.open_array(
    "acquisitions.zarr/data",
    shape=(365, 4096),
    chunks=[[31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31], [4096]],
    dtype="float32",
)
# Each month gets its own chunk — no padding for short months
```

<v-clicks>

- **Status**: experimental in zarr-python 3.2 (ZEP 3 draft)
- Stabilization expected in **zarr-python 3.3**
- ⚠️ Data written with rectilinear chunks is **not readable by older Zarr versions**

</v-clicks>
```

- [ ] **Step 6: Commit**

```bash
git add slides.md
git commit -m "feat: add chunking strategy and rectilinear chunking slides (section 4)"
```

---

### Task 6: Write Slides — Section 5 (Icechunk)

**Files:**
- Modify: `slides.md`

- [ ] **Step 1: Add the concurrent writes problem slide (Slide 17)**

Append to `slides.md`:

```md
---
layout: default
---

# The Concurrent Write Problem

Multiple instruments writing to the same Zarr dataset in cloud storage:

<v-clicks>

- **No transactions**: partial writes leave arrays inconsistent
- **No versioning**: can't roll back a bad acquisition
- **No audit trail**: which instrument wrote which data, and when?
- **Race conditions**: two writers updating the same chunk = last write wins

</v-clicks>

<div v-click class="mt-4 p-4 bg-red-50 rounded border border-red-200 text-red-800">
Zarr v2/v3 storage specs don't define concurrency semantics — it's up to you.
</div>
```

- [ ] **Step 2: Add the Icechunk intro slide with diagram (Slide 18)**

```md
---
layout: default
---

# Icechunk: Transactional Storage for Zarr

<Excalidraw drawFilePath="/diagrams/icechunk-architecture.excalidraw" :darkMode="false" class="w-full h-70" />

- Rust core, Python wrapper — **no external database** required
- All state lives in object storage (S3, GCS, Azure, local)
```

- [ ] **Step 3: Add the Icechunk features slide (Slide 19)**

```md
---
layout: default
---

# Icechunk Key Features

<v-clicks>

- **Serializable transaction isolation**
  - Reads use committed snapshots
  - Writes commit atomically — all or nothing
- **Git-like version control**
  - Branches: `main`, `experiment-1`, `staging`
  - Tags: immutable release markers
  - Time-travel: read any historical snapshot
- **Virtual datasets**
  - Store chunk references ("pointers") to existing HDF5/NetCDF/GRIB
  - Update metadata without copying underlying data

</v-clicks>
```

- [ ] **Step 4: Add the Icechunk integration code slide (Slide 20)**

```md
---
layout: default
---

# Icechunk with zarr-python

```python {all|1-5|7-9|11-14}
import icechunk
import zarr

# Open (or create) a repository in S3
repo = icechunk.Repository.open_or_create(
    storage=icechunk.s3_storage(bucket="caur-data", prefix="tomography/"),
)

# Get a writable session on the main branch
session = repo.writable_session("main")
store = session.store

# Use it exactly like any Zarr v3 store
root = zarr.open_group(store=store)
root.create_array("waveforms", shape=(10000, 4096), dtype="float32")

# Commit atomically
session.commit("Add waveform array from instrument A")
```

`IcechunkStore` is a **drop-in Zarr v3 store** — existing code works unchanged.
```

- [ ] **Step 5: Add the Icechunk for tomography slide (Slide 21)**

```md
---
layout: default
---

# Icechunk for Tomography Pipelines

<v-clicks>

- **Multi-instrument safety**: each instrument opens its own session — no race conditions
- **Rollback**: bad acquisition? Revert to the last good snapshot
- **Branch per experiment**: isolate experimental writes, merge when validated
- **Time-travel**: reproduce any analysis from any point in history
- **Virtual references**: point to existing data files without copying — migrate incrementally

</v-clicks>

```python
# Roll back to a known good state
repo.reset_branch("main", snapshot_id="abc123")

# Branch for an experiment
session = repo.writable_session("experiment-42")
```
```

- [ ] **Step 6: Commit**

```bash
git add slides.md
git commit -m "feat: add Icechunk slides (section 5)"
```

---

### Task 7: Write Slides — Sections 6-7 (v2→v3 Migration, Wrap-up)

**Files:**
- Modify: `slides.md`

- [ ] **Step 1: Add the v2 vs v3 comparison slide (Slide 22)**

Append to `slides.md`:

```md
---
layout: default
---

# v2 → v3 At a Glance

| | **Zarr v2** | **Zarr v3** |
|---|---|---|
| **Metadata** | `.zarray` + `.zattrs` (separate) | Single `zarr.json` |
| **Codecs** | `compressor` + `filters` (flat) | Typed, ordered pipeline |
| **Endianness** | Baked into dtype | Explicit in `BytesCodec` |
| **Sharding** | Not supported | First-class `ShardingCodec` |
| **Chunking** | Uniform only | Uniform + rectilinear |
| **Extensions** | Limited | Extensible codecs, stores, dtypes |

<v-click>

**Migration**: zarr-python 3.x **reads v2 data natively** — migrate incrementally, not all at once.

</v-click>
```

- [ ] **Step 2: Add the recommendations slide (Slide 23)**

```md
---
layout: default
---

# Recommendations

<v-clicks>

1. **Use sharding** for cloud writes — eliminate small-object overhead
2. **Evaluate zarrs-python** for write-heavy local pipelines
3. **Adopt Icechunk** for multi-writer safety and versioning
4. **Align chunk shapes** to your dominant access pattern (80% rule)
5. **Explore rectilinear chunking** if acquisition windows are variable-length
6. **Start with zarr-python 3.x** — it reads v2 data, so migration is incremental

</v-clicks>
```

- [ ] **Step 3: Add the resources slide (Slide 24)**

```md
---
layout: end
---

# Resources

- **Zarr**: [zarr.dev](https://zarr.dev) — spec, docs, ZEPs
- **Icechunk**: [icechunk.io](https://icechunk.io) — docs, getting started
- **zarrs-python**: [github.com/zarrs/zarrs-python](https://github.com/zarrs/zarrs-python)
- **ZEP 2**: Sharding spec
- **ZEP 3**: Rectilinear chunking spec (draft)
- **zarr-python 3.x**: [zarr.readthedocs.io](https://zarr.readthedocs.io)
```

- [ ] **Step 4: Commit**

```bash
git add slides.md
git commit -m "feat: add v2-v3 migration and wrap-up slides (sections 6-7)"
```

---

### Task 8: Verify Presentation

**Files:**
- Read: `slides.md` (full review)

- [ ] **Step 1: Start the dev server and verify all slides render**

Run:
```bash
cd /Users/seanharkins/projects/zarr_presentation
npx slidev --open false
```

Expected: Server starts without errors. Navigate to `http://localhost:3030` and click through all 24 slides. Verify:
- Title slide renders with name and date
- All four Excalidraw diagrams load and display
- Code blocks have Python syntax highlighting
- v-clicks animate on click
- Two-column layouts render correctly
- Comparison table on slide 22 is readable

- [ ] **Step 2: Check slide count**

Confirm there are 24 slides total by navigating to the last slide. The slide counter should show `24/24`.

- [ ] **Step 3: Fix any rendering issues**

If any slides have layout problems, missing diagrams, or broken syntax, fix them.

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "fix: resolve any rendering issues from review"
```

Only create this commit if changes were made in step 3.
