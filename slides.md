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

Today: what the **v3 ecosystem** solves for high-throughput, cloud-targeted instrument pipelines:

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

<div class="flex flex-col items-center">
  <div class="w-3/4">
    <Excalidraw drawFilePath="/diagrams/sharding-concept.excalidraw" :darkMode="false" />
  </div>
</div>

<div class="mt-4 text-sm">

- **Inner chunks**: fine-grained for partial reads
- **Shard (outer chunk)**: one storage object containing many inner chunks
- **Shard index**: byte offsets enable reading a single inner chunk without fetching the whole shard

</div>

---
layout: default
---

# Sharding in zarr-python 3.x

```python {all|3-8|4|5-8}
import zarr
from zarr.codecs import BloscCodec, BytesCodec

arr = zarr.create_array(
    "s3://bucket/tomography.zarr/data",
    shape=(10000, 4096, 4096),
    shards=(100, 512, 512),          # shard (outer) shape -> 1 storage object
    chunks=(10, 32, 32),             # inner chunk shape -> partial reads
    dtype="float32",
    serializer=BytesCodec(),
    compressors=[BloscCodec(cname="zstd", clevel=3)],
)
```

Each shard: `(100×512×512)` = **~100 MB** — one S3 object  
Inner chunks: `(10×32×32)` — partial reads at fine granularity

---
layout: default
---

# Sharding Trade-offs

<div class="grid grid-cols-2 gap-8 mt-4 text-gray-600">
<div>

**Benefits**

- Drastically reduces overall object count
- Lower cloud API costs
- Better sequential throughput (via coalesce)
- Shard index enables partial reads

</div>
<div>

**Considerations**

- **Write amplification**: updating one inner chunk rewrites the shard
- Shard size sweet spot: **100 MB** (but fully dependent on storage backend)
- Too large → Higher overhead during writing when a shard is rewritten
- Best when write pattern is **append-heavy** (not random updates)

</div>
</div>

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
zarr.create_array(
    ...,
    compressor=Blosc(cname="zstd", clevel=3),
    filters=[Delta(dtype="float32")],
)
```

<v-clicks>

- Filter vs. compressor distinction is **arbitrary**
- No checksums in the pipeline

</v-clicks>

---
layout: default
---

# v3: Typed, Ordered Codec Pipeline

<div class="flex flex-col items-center">
  <div class="w-3/4">
    <Excalidraw drawFilePath="/diagrams/codec-pipeline.excalidraw" :darkMode="false" />
  </div>
</div>

<div class="mt-4 text-sm">

| Stage | Role | Examples |
|-------|------|----------|
| **Array → Array** | Transform array data | `TransposeCodec`, delta, scaling |
| **Array → Bytes** | Serialize to bytes | `BytesCodec` (explicit endianness) |
| **Bytes → Bytes** | Compress / checksum | `BloscCodec`, `ZstdCodec`, `Crc32cCodec` |

Codecs execute in **declared order** — no ambiguity. v3 formalizes the idea that
the in-memory array representation is an implementation specific detail.  Only
the Array->Bytes codec defines explicit representation. 

</div>

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

```bash
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
layout: default
---

# zarrs-python: Performance & Caveats

<div class="grid grid-cols-2 gap-8 mt-4 text-gray-600">
<div>

**Where it shines**

- Write-heavy workloads: biggest speedup
- Sharded data: parallelizes inner chunk codec work
- Large contiguous reads
- Configurable thread pool per operation

</div>
<div>

**Current limitations**

- Best for write pipelines, post-acquisition processing
- v0.2.3 — API is still stabilizing
- Falls back to Python pipeline for unsupported stores

</div>
</div>

<!--
For Caur Tech: if instruments write to local disk first then sync to cloud,
zarrs-python can accelerate the local write path significantly.
-->

---
layout: default
---

# Chunking Is Hard

Optimal chunking is driven by **end-user access patterns** — there's no universal answer.

But there are clear guidelines on what **not** to do:

<v-clicks>

- Don't use chunks smaller than **1 MB** — HTTP overhead dominates
- Don't use chunks larger than **100 MB** — wasted bandwidth on partial reads
- Don't ignore your dominant query pattern — chunking against it means full-array scans
- Don't create deep group hierarchies — each level adds LIST call overhead
- Don't skip consolidated metadata — per-array metadata fetches add up fast

</v-clicks>

<div v-click class="mt-6 p-4 bg-blue-50 rounded border border-blue-200">

For a thorough guide on chunking strategies and datacube design, see the **Development Seed Datacube Guide**: [developmentseed.org/datacube-guide](https://developmentseed.org/datacube-guide/latest/index.html)

</div>

---
layout: default
---

# Rectilinear Chunking

<div class="flex flex-col items-center">
  <div class="w-3/4">
    <Excalidraw drawFilePath="/diagrams/rectilinear-chunking.excalidraw" :darkMode="false" />
  </div>
</div>

<div class="mt-4 text-sm">

- Per-dimension chunk size **lists** instead of uniform integers
- No more padding or artificial splitting for variable-length data

</div>

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

arr = zarr.create_array(
    "acquisitions.zarr/data",
    shape=(365, 4096),
    chunks=[[31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31], [4096]],
    dtype="float32",
)
# Each month gets its own chunk — no padding for short months
```

<v-clicks>

- **Status**: experimental in zarr-python 3.2
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

<div class="flex flex-col items-center">
  <div class="w-3/4">
    <Excalidraw drawFilePath="/diagrams/icechunk-architecture.excalidraw" :darkMode="false" />
  </div>
</div>

<div class="mt-4 text-sm">

- Rust core, Python wrapper — **no external database** required
- All state lives in object storage (S3, GCS, Azure, local)

</div>

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
