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

---
layout: center
---

# You Know Zarr v2

Today: what the **v3 ecosystem** solves for high-throughput, cloud-targeted instrument pipelines

<v-clicks>

- Small-object overhead → **Sharding**
- Codec rigidity & Python overhead → **v3 Codec Pipeline + zarrs-python**
- Chunking inflexibility → **Rectilinear Chunks**
- Concurrent write safety → **Icechunk**

</v-clicks>

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

---
layout: default
---

# Sharding: Decouple Chunks from Objects

<Excalidraw drawFilePath="/diagrams/sharding-concept.excalidraw" :darkMode="false" class="w-full h-80" />

- **Inner chunks**: fine-grained for partial reads
- **Shard (outer chunk)**: one storage object containing many inner chunks
- **Shard index**: byte offsets enable reading a single inner chunk without fetching the whole shard

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

---
layout: default
---

# Rectilinear Chunking (ZEP 3)

<Excalidraw drawFilePath="/diagrams/rectilinear-chunking.excalidraw" :darkMode="false" class="w-full h-70" />

Per-dimension chunk size **lists** instead of uniform integers.
No more padding or artificial splitting for variable-length data.

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
- Data written with rectilinear chunks is **not readable by older Zarr versions**

</v-clicks>

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

---
layout: default
---

# Icechunk: Transactional Storage for Zarr

<Excalidraw drawFilePath="/diagrams/icechunk-architecture.excalidraw" :darkMode="false" class="w-full h-70" />

- Rust core, Python wrapper — **no external database** required
- All state lives in object storage (S3, GCS, Azure, local)

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
