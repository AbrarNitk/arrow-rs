# Apache Arrow Deep Dive In `arrow-rs`

This directory is a ground-up, implementation-linked tutorial for understanding Apache Arrow in the `arrow-rs` repository at a level where you should be able to:

1. explain the Arrow memory format from machine constraints upward,
2. justify why the layout uses buffers, bitmaps, offsets, and child arrays,
3. answer low-level questions about nullability, slicing, alignment, IPC, FFI, and row conversion,
4. follow the `arrow-rs` code with confidence,
5. sketch or build another Arrow implementation from scratch.

The earlier version of these notes explained the crate structure correctly, but still leaned too much toward module summary. This version is meant to read like a systems tutorial: every major abstraction is presented as the answer to a specific storage, safety, or execution problem.

## The questions this tutorial is trying to answer

If you cannot answer these questions, you do not yet understand Arrow deeply enough:

- Why does Arrow store data in buffers instead of per-value objects?
- Why is nullability carried by a bitmap instead of by tagged values?
- Why are offsets the right answer for variable-width data?
- Why is `ArrayData` the critical invariant boundary?
- Why are schema and physical layout separated into different crates?
- Why is `Array` an `unsafe trait`?
- Why does IPC preserve the in-memory layout instead of inventing a separate body format?
- Why does Arrow need a row-oriented derivative format at all if it is columnar?
- Why does the Rust implementation deliberately avoid a built-in `ChunkedArray` abstraction?

The reading order is designed to answer those in dependency order.

## Recommended reading order

1. [01-why-arrow-and-crate-layering.md](./01-why-arrow-and-crate-layering.md)
2. [02-memory-buffers-and-alignment.md](./02-memory-buffers-and-alignment.md)
3. [03-arraydata-and-physical-layout.md](./03-arraydata-and-physical-layout.md)
4. [04-schema-fields-and-logical-types.md](./04-schema-fields-and-logical-types.md)
5. [05-arrays-nullability-and-record-batches.md](./05-arrays-nullability-and-record-batches.md)
6. [06-variable-width-layouts-and-views.md](./06-variable-width-layouts-and-views.md)
7. [07-ffi-and-zero-copy-boundaries.md](./07-ffi-and-zero-copy-boundaries.md)
8. [08-ipc-files-streams-and-message-layout.md](./08-ipc-files-streams-and-message-layout.md)
9. [09-row-format-simd-and-execution-patterns.md](./09-row-format-simd-and-execution-patterns.md)
10. [10-arrow-and-parquet-integration.md](./10-arrow-and-parquet-integration.md)
11. [11-utilities-and-application-patterns.md](./11-utilities-and-application-patterns.md)
12. [12-worked-example-from-arrays-to-ipc.md](./12-worked-example-from-arrays-to-ipc.md)

## How to read these notes

Each chapter has a `Source markers` section. Use them in this order:

1. read the conceptual explanation first,
2. check the diagram until you can restate it in your own words,
3. only then open the linked code and map the concepts onto concrete types and functions.

If you open the code too early, Arrow can look like “a lot of array wrappers and builders.” In reality it is a tight answer to a small set of hard problems:

- represent heterogeneous tabular data compactly,
- make scans cache-friendly,
- make slicing zero-copy,
- preserve logical nullability without destroying physical density,
- allow safe high-level APIs on top of low-level raw buffers,
- serialize and share that memory across libraries and languages.

## Mental model of the whole system

At the highest level, Arrow is a contract between:

- a producer that arranges data into a valid columnar memory layout,
- and a consumer that can interpret those bytes as typed arrays without row-by-row decoding.

```text
application values
   │
   ├─ choose logical schema
   ├─ arrange bytes into Arrow buffers
   ├─ attach null bitmaps / offsets / child arrays
   ├─ validate or trust the layout
   └─ expose typed arrays / batches
        │
        ▼
      Arrow memory
        │
        ├─ zero-copy slicing
        ├─ compute kernels
        ├─ IPC serialization
        ├─ FFI export
        └─ Parquet bridge
```

That is the whole system in one picture.

## Core source map

- Workspace and crate boundaries:
  [`Cargo.toml`](../../Cargo.toml),
  [`arrow/Cargo.toml`](../../arrow/Cargo.toml),
  [`arrow/src/lib.rs`](../../arrow/src/lib.rs)
- Low-level memory and buffers:
  [`arrow-buffer/src/lib.rs`](../../arrow-buffer/src/lib.rs),
  [`arrow-buffer/src/alloc/alignment.rs`](../../arrow-buffer/src/alloc/alignment.rs),
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs),
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs),
  [`arrow-buffer/src/buffer/null.rs`](../../arrow-buffer/src/buffer/null.rs),
  [`arrow-buffer/src/buffer/offset.rs`](../../arrow-buffer/src/buffer/offset.rs)
- Physical array data:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs),
  [`arrow-data/src/byte_view.rs`](../../arrow-data/src/byte_view.rs),
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs)
- Logical schema:
  [`arrow-schema/src/datatype.rs`](../../arrow-schema/src/datatype.rs),
  [`arrow-schema/src/field.rs`](../../arrow-schema/src/field.rs),
  [`arrow-schema/src/schema.rs`](../../arrow-schema/src/schema.rs),
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs)
- Typed arrays and batches:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs),
  [`arrow-array/src/array/mod.rs`](../../arrow-array/src/array/mod.rs),
  [`arrow-array/src/array/byte_array.rs`](../../arrow-array/src/array/byte_array.rs),
  [`arrow-array/src/array/byte_view_array.rs`](../../arrow-array/src/array/byte_view_array.rs),
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs)
- IPC and row format:
  [`arrow-ipc/src/lib.rs`](../../arrow-ipc/src/lib.rs),
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs),
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs),
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs)
- Arrow + Parquet integration:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs),
  [`parquet/src/arrow/schema/mod.rs`](../../parquet/src/arrow/schema/mod.rs)
- Examples:
  [`arrow/examples`](../../arrow/examples),
  [`parquet/examples`](../../parquet/examples)
