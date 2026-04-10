# Apache Arrow Deep Dive In `arrow-rs`

This directory is a guided, code-first tutorial for understanding Apache Arrow in the `arrow-rs` repository. It is written in the order you should read the implementation if your goal is to understand:

1. why Arrow is designed around columnar, zero-copy memory,
2. how `arrow-rs` splits responsibilities across crates,
3. how buffers, bitmaps, offsets, and child arrays physically represent data,
4. how logical schema and physical layout are connected,
5. how record batches, IPC, FFI, row format, and Parquet integration sit on top.

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

## How to use these notes

Each chapter has a `Source markers` section. Those markers point at the main implementation entry points in this repository. Read the prose first, then open the linked code and follow the control flow outward only when you need more detail.

The sequence is intentional:

- start from machine-level memory and alignment assumptions,
- then understand the low-level buffer contracts,
- then study `ArrayData`, which is the physical invariant boundary,
- then study schema and high-level typed arrays,
- then move upward into `RecordBatch`, FFI, IPC, row encoding, and Parquet.

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
