# Zarr v3 in Practice — Presentation Design Spec

## Overview

- **Title:** "Zarr v3 in Practice: Sharding, Codecs, and Icechunk for High-Throughput Instrument Data"
- **Duration:** 20 minutes
- **Format:** Slidev (markdown-based slides), static code snippets only (no live demos)
- **Audience:** Engineers at Caur Tech (https://caurtech.com/en/) — Zarr v2 power users who build ambient noise tomography instruments that write data directly to Zarr
- **Approach:** Problem-driven — each section frames a pain point the audience likely experiences, then introduces the Zarr v3 / ecosystem feature that solves it
- **Estimated slide count:** ~24 slides

## Audience Assumptions

- Comfortable with Zarr v2 internals: chunk layout, compression tuning, object-store access patterns
- Writing high-frequency instrument data directly to Zarr in cloud or hybrid environments
- Have not yet migrated to Zarr v3
- May have multiple instruments / processes writing concurrently

## Slide-by-Slide Outline

### Section 1 — Title & Context (1 min, ~2 slides)

**Slide 1: Title**
- "Zarr v3 in Practice: Sharding, Codecs, and Icechunk for High-Throughput Instrument Data"
- Speaker name, date

**Slide 2: Framing**
- "You know Zarr v2. Here's what the v3 ecosystem solves for high-throughput, cloud-targeted instrument pipelines."
- Brief agenda overview

### Section 2 — The Small-Object Problem → Sharding (4 min, ~4 slides)

**Slide 3: The Problem**
- High-frequency instrument writes create millions of tiny objects in S3
- S3 LIST/GET costs, latency, 404-consistency issues, metadata overhead
- Each logical chunk = one storage object in v2

**Slide 4: Sharding Concept**
- Excalidraw diagram: logical chunk grid packed into larger shard objects
- Inner chunks vs. shard (outer) chunks
- Shard index enables partial reads of individual inner chunks without fetching the whole shard

**Slide 5: Sharding in Code**
- Code snippet: configuring `ShardingCodec` in zarr-python 3.x
- Example: inner chunk shape `[32, 32]`, shard shape `[512, 512]` = 256 inner chunks per shard

**Slide 6: Sharding Trade-offs**
- Shard size vs. partial-read granularity
- Write amplification when updating single inner chunks within a shard
- Sweet spot guidance: target 1-100 MB per shard object
- When NOT to shard (small datasets, random single-chunk updates)

### Section 3 — Compression Overhead → Codec Pipeline & zarrs-python (4 min, ~5 slides)

**Slide 7: The Problem**
- v2's flat `compressor` + `filters` model is rigid and ordering is implicit
- Python codec overhead hurts on large write workloads

**Slide 8: v3 Codec Pipeline**
- Typed, ordered pipeline with three stages:
  - Array-to-array: `TransposeCodec`, delta encoding, scaling
  - Array-to-bytes: `BytesCodec` (explicit endianness — v3 dtypes no longer carry endianness)
  - Bytes-to-bytes: `BloscCodec`, `ZstdCodec`, `GzipCodec`, `Crc32cCodec`
- Excalidraw diagram: flow from raw array through each codec stage to stored bytes

**Slide 9: Codec Pipeline in Code**
- Code snippet: composing a multi-stage codec pipeline in zarr-python 3.x
- Example: `TransposeCodec(order=(1,0))` → `BytesCodec()` → `BloscCodec(cname='zstd', clevel=3)`

**Slide 10: zarrs-python — Rust-Powered Codecs**
- What it is: drop-in `CodecPipeline` replacement backed by the Rust `zarrs` crate via PyO3
- Install: `pip install zarrs`
- Enable: `zarr.config.set({"codec_pipeline.path": "zarrs.ZarrsCodecPipeline"})`
- Configurable concurrency: `chunk_concurrent_maximum`, `chunk_concurrent_minimum`

**Slide 11: zarrs-python Performance & Caveats**
- Biggest gains on writes and sharded data
- Supports `validate_checksums` and `direct_io` options
- Limitation: filesystem stores only — no remote/cloud stores yet
- Best for: local write-heavy pipelines, post-acquisition processing

### Section 4 — Cloud Performance → Chunking, Access Patterns & Rectilinear Chunks (4 min, ~5 slides)

**Slide 12: Chunking Strategy**
- Align chunks to dominant access patterns
- Tomography context: time-series slicing vs. spatial/frequency queries
- Avoid chunks that are too small (object overhead) or too large (wasted reads)

**Slide 13: Object-Store Best Practices**
- Chunk size sweet spot: 1-100 MB per object
- Consolidated metadata to avoid per-array metadata fetches
- Avoid deep group hierarchies (each level = additional LIST call)
- Read coalescing and concurrent reads

**Slide 14: Rectilinear Chunking (ZEP 3)**
- What it is: per-dimension chunk size lists instead of uniform integers
- Excalidraw diagram: uniform chunks vs. variable-sized chunks along an axis

**Slide 15: Rectilinear Chunking — Use Cases**
- Variable-length acquisition windows from instruments
- Virtualizing heterogeneous NetCDF/HDF5 collections (files with different sizes along concat axis)
- Run-length encoding support in metadata for efficiency

**Slide 16: Rectilinear Chunking in Code**
- Feature flag: `zarr.config.set({"array.rectilinear_chunks": True})`
- Configuration: `chunks=[[31, 28, 31, 30], [256, 256]]`
- Status: experimental in zarr-python 3.2, stabilization expected in 3.3
- Caveat: data not readable by older Zarr versions

### Section 5 — Data Integrity & Collaboration → Icechunk (5 min, ~5 slides)

**Slide 17: The Problem**
- Concurrent instrument writes to cloud storage risk corruption
- No versioning, no rollback, no audit trail
- Partial writes can leave arrays in inconsistent state

**Slide 18: What Icechunk Is**
- Transactional storage engine for Zarr v3
- Rust core with Python wrapper
- All state lives in object storage — no external database or catalog required
- Excalidraw diagram: architecture showing writers, branches, snapshots in object storage

**Slide 19: Key Features**
- Serializable transaction isolation: reads use committed snapshots, writes commit atomically
- Git-like version control: branches (dev/stage/prod), tags (immutable markers), time-travel to any snapshot
- Virtual datasets: store chunk references to existing HDF5/NetCDF/GRIB files without copying data

**Slide 20: Icechunk Integration**
- `IcechunkStore` is a drop-in Zarr v3 store
- Code snippet:
  ```python
  import icechunk
  repo = icechunk.Repository.open(...)
  store = repo.writable_session("main").store
  root = zarr.open(store=store)
  ```

**Slide 21: Icechunk for Tomography Pipelines**
- Multiple instruments writing to the same cloud dataset safely
- Rollback if a bad acquisition corrupts data
- Branch per instrument or per experiment, merge when validated
- Time-travel for reproducible analysis

### Section 6 — v2 → v3 At a Glance (1 min, ~1 slide)

**Slide 22: Comparison Table**

| Aspect | v2 | v3 |
|---|---|---|
| Metadata | `.zarray` + `.zattrs` (separate files) | Single `zarr.json` |
| Codecs | `compressor` + `filters` (flat) | Typed, ordered pipeline |
| Data types | Endianness in dtype | Endianness in `BytesCodec` |
| Sharding | Not supported | First-class via `ShardingCodec` |
| Chunking | Uniform only | Uniform + rectilinear (experimental) |
| Extensions | Limited | Extensible codec/store/dtype system |

- Note: zarr-python 3.x reads v2 data natively; incremental migration is possible

### Section 7 — Wrap-up & Resources (1 min, ~2 slides)

**Slide 23: Recommendations**
- Use sharding for cloud writes to eliminate small-object overhead
- Evaluate zarrs-python for write-heavy local pipelines
- Adopt Icechunk for multi-writer safety and versioning
- Align chunk shapes to your dominant access patterns
- Explore rectilinear chunking if acquisition windows are variable-length

**Slide 24: Resources**
- zarr.dev — spec, docs, ZEPs
- icechunk.io — docs, getting started
- github.com/zarrs/zarrs-python — Rust codec pipeline
- ZEP 2 (sharding), ZEP 3 (rectilinear chunking)

## Diagrams (Excalidraw)

Four Excalidraw diagrams to create:

1. **Sharding concept** (Slide 4): Logical chunk grid → packed shard objects with index. Shows how inner chunks map into a shard and how the index enables partial reads.
2. **Codec pipeline flow** (Slide 8): Raw array → array-to-array → array-to-bytes → bytes-to-bytes → stored bytes. Each stage labeled with example codecs.
3. **Rectilinear vs. uniform chunking** (Slide 14): Side-by-side comparison of a 2D array chunked uniformly vs. with variable chunk sizes along one axis.
4. **Icechunk architecture** (Slide 18): Multiple writer processes committing to branches, snapshots stored in object storage, no external DB. Shows transaction isolation.

Style: clean, schematic — boxes, arrows, labels. No decorative elements.

## Slidev Configuration

- **Theme:** `seriph` (clean, technical aesthetic)
- **Code highlighting:** Shiki with Python syntax
- **Excalidraw:** `@slidev/plugin-excalidraw` or embedded `.excalidraw` files
- **Transitions:** Simple fade
- **Export:** PDF-capable for post-talk sharing
