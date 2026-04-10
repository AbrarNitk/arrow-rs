# 11. Utilities, Common Workflows, And How To Apply Arrow

The earlier chapters focused on invariants and storage. This chapter answers the practical question: how do you use these layers in real code without losing the underlying model?

## Workflow 1: Build arrays directly

Use typed array constructors or builders when:

- data is already in process memory,
- you want exact control over nulls and offsets,
- you are writing kernels or tests.

Example shape:

```rust
use arrow_array::{Int32Array, StringArray, RecordBatch};
use std::sync::Arc;

let ids = Int32Array::from(vec![1, 2, 3]);
let names = StringArray::from(vec![Some("alice"), None, Some("carol")]);

let batch = RecordBatch::try_from_iter(vec![
    ("id", Arc::new(ids) as _),
    ("name", Arc::new(names) as _),
])?;
# Ok::<(), arrow_schema::ArrowError>(())
```

Use this path when you care most about memory layout and low overhead.

## Workflow 2: Build `ArrayData` when implementing lower-level systems

Use `ArrayData` or `ArrayDataBuilder` when:

- importing foreign memory,
- writing specialized readers,
- constructing arrays from raw buffers,
- validating layout boundaries explicitly.

This is the right layer for custom IPC/FFI bridges and low-level engine internals.

## Workflow 3: Use `RecordBatch` as the pipeline unit

Use `RecordBatch` when:

- moving tabular data between operators,
- writing readers/writers,
- feeding Parquet or IPC,
- integrating with query engines.

This is the normal application-level Arrow unit.

## Workflow 4: IPC for transport and caching

Use `arrow-ipc` when:

- sending Arrow data over the network,
- writing fast reopenable caches,
- preserving Arrow layout semantics across process boundaries,
- enabling zero-copy reads from memory-mapped files.

The file variant is better for persisted artifacts. The stream variant is better for forward-only transport.

## Workflow 5: FFI for language or library boundaries

Use the C Data Interface when:

- another library already understands Arrow C ABI,
- you want to avoid full serialization,
- schema and buffers can stay live across the boundary.

This is often the lowest-overhead interoperability path.

## Workflow 6: Row conversion for sort/group heavy operators

Use `arrow-row` when:

- comparing composite keys repeatedly,
- building sort keys,
- implementing grouping or window pre-processing,
- lexicographic row comparison is the hot path.

Do not confuse this with replacing Arrow storage. It is a derivative execution format, not the primary storage format.

## Workflow 7: Parquet for persistence

Use Parquet when:

- data must be stored on disk efficiently,
- compression and encoded pages matter,
- row groups and statistics matter,
- Arrow is the in-memory model but Parquet is the archival or exchange artifact.

Common pattern:

```text
source data
  -> Arrow arrays / RecordBatch
  -> compute / transform
  -> Parquet for storage

later:
Parquet
  -> Arrow RecordBatch
  -> compute / filtering / analytics
```

## Examples worth reading in this repository

- zero-copy IPC reading:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs)
- Parquet Arrow reading/writing examples:
  [`parquet/examples`](../../parquet/examples)

## A practical reading path through the code

If your goal is to become productive in `arrow-rs`, read in this order:

1. `arrow-buffer`
2. `arrow-data`
3. `arrow-schema`
4. `arrow-array`
5. `arrow-ipc`
6. `arrow-row`
7. `parquet/src/arrow`

That order follows dependency and concept flow. It is the shortest path from "why is this buffer shaped like this?" to "how does this become a Parquet file or IPC stream?"

## Source markers

- High-level facade and examples:
  [`arrow/src/lib.rs`](../../arrow/src/lib.rs)
- Typed array usage patterns:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs)
- Batch API:
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs)
- IPC example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs)
- Row conversion:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs)
- Parquet bridge:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs)
