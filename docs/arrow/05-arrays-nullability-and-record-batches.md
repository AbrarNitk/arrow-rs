# 5. Typed Arrays, Nullability, And `RecordBatch`

Now we move from the generic physical representation to the user-facing array layer.

## The `Array` trait is the type-erased boundary

`arrow-array/src/array/mod.rs` defines `unsafe trait Array`.

The `unsafe` is important. It means a type implementing `Array` must uphold layout and semantic invariants expected by the rest of the system. The docs explicitly warn that third-party implementations are easy to get wrong and may become sealed in the future.

That tells you how central invariants are in this codebase: dynamic polymorphism is supported, but not casually.

## Two access styles coexist on purpose

Arrow Rust supports:

1. concrete typed arrays such as `Int32Array`, `StringArray`, `ListArray`,
2. type-erased `dyn Array` / `ArrayRef`.

Why both?

- typed arrays give fast, specialized access,
- `dyn Array` supports heterogeneous schemas and generic operators,
- downcasting lets high-level code stay generic while kernels recover concrete views when needed.

## Nullability is separate from value storage

For fixed-width arrays like `Int32Array`, nulls do not remove slots from the values buffer.

```text
values:   [10][99][30][40]
validity:  1   0   1   1

logical array:
  0 -> 10
  1 -> null
  2 -> 30
  3 -> 40
```

This is why array docs often warn that `values()` may contain arbitrary but valid bytes at null positions. The bitmap carries the truth; the physical value slot only needs to be well-formed enough for safe access patterns.

## Primitive arrays are the cleanest example

`PrimitiveArray<T>` is the ideal Arrow shape:

- optional `NullBuffer`
- dense `ScalarBuffer<T::Native>`

That is why numeric kernels are often the easiest place to see Arrow's performance model. The runtime work is mostly:

- scan bitmap if nulls exist,
- scan dense native values,
- apply operation with minimal dispatch.

## Variable-width arrays add offsets, not objects

`GenericByteArray` stores:

- `NullBuffer`
- `OffsetBuffer<i32>` or `OffsetBuffer<i64>`
- values `Buffer`

This keeps string and binary arrays structurally close to primitive arrays: still just a handful of buffers, still sliceable, still zero-copy where possible.

## Why `RecordBatch` is the unit of tabular work

A `RecordBatch` is:

- one schema,
- N columns,
- all columns same logical row count.

```text
RecordBatch
  schema: [id: Int32, name: Utf8, score: Float64]
  columns:
    col0 len=3
    col1 len=3
    col2 len=3

rows:
  row0 = (col0[0], col1[0], col2[0])
  row1 = (col0[1], col1[1], col2[1])
  row2 = (col0[2], col1[2], col2[2])
```

Arrow does not store rows physically here. The rows are reconstructed by aligning array positions across columns.

`RecordBatch` is useful because it is:

- large enough to amortize per-batch overhead,
- small enough to stream and pipeline,
- neutral across compute, IPC, Parquet, CSV, JSON, and query engines.

## Why there is no Rust `ChunkedArray`

The crate docs explicitly choose not to implement a `ChunkedArray` abstraction. Instead they recommend:

- `Vec<ArrayRef>`
- iterators
- streams

That decision fits Rust well:

- composition is simpler,
- async and lazy pipelines are more natural,
- there is less pressure to create a large, opinionated mid-layer abstraction.

At the batch level, this means a dataset is usually represented as `Vec<RecordBatch>` or a `RecordBatchReader`/stream.

## Reader and writer traits

`RecordBatchReader` and `RecordBatchWriter` define a generic tabular IO boundary:

- readers yield batches with a stable schema,
- writers consume batches and usually finalize on `close`.

These traits are what let IPC, Parquet, and downstream engines plug into the same tabular contract.

## Source markers

- `Array` trait and safety boundary:
  [`arrow-array/src/array/mod.rs`](../../arrow-array/src/array/mod.rs#L100)
- `arrow-array` crate overview and `ChunkedArray` policy:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L1),
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L164)
- Variable-width arrays:
  [`arrow-array/src/array/byte_array.rs`](../../arrow-array/src/array/byte_array.rs#L87)
- `RecordBatch`, reader, and writer traits:
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L30),
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L45),
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L224)
