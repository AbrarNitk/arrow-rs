# 1. Why Arrow Exists And How `arrow-rs` Is Layered

Apache Arrow is not primarily a file format. It is a memory format and an interoperability contract for tabular and nested data. The main promise is:

- keep data in a columnar layout that CPUs can scan efficiently,
- represent nullability and variable-width values without per-value boxing,
- share memory across libraries and languages without copying,
- serialize that in-memory model to IPC or other formats when needed.

That is why this tutorial starts below `RecordBatch` and below schema conversion. The important design decision is not "what APIs exist?" but "what invariants must hold so that one piece of memory can safely pretend to be many typed arrays?"

## The mental model

Arrow uses a layered model:

```text
Application / query engine / utility
            │
            ▼
Typed arrays, builders, RecordBatch
            │
            ▼
Logical schema and field metadata
            │
            ▼
Generic physical representation: ArrayData
            │
            ▼
Shared aligned buffers, bitmaps, offsets
            │
            ▼
OS allocation / mmap / foreign memory / network buffers
```

The lower layers know nothing about SQL, CSV, Parquet, or query planning. They only know memory layout.

## Why the repository is split into many Arrow crates

The workspace deliberately separates concerns:

- `arrow-buffer`: aligned memory, buffers, bitmaps, offsets, native value containers.
- `arrow-data`: generic physical array representation and validation.
- `arrow-schema`: logical types, fields, schemas, metadata, and schema FFI.
- `arrow-array`: typed arrays, builders, `RecordBatch`, and trait-object access.
- `arrow-ipc`: Arrow stream/file serialization.
- `arrow-row`: row-oriented comparable encoding built from Arrow arrays.
- `arrow`: a convenience facade that re-exports the layers most users want.

This is not just crate hygiene. It reflects the architecture:

- low-level code must be reusable without dragging in all high-level APIs,
- schema should exist independently of physical buffers,
- IPC and Parquet must sit on top of the in-memory model instead of owning it,
- query engines can choose only the crates they need.

## Why the top-level `arrow` crate is mostly a facade

The `arrow` crate documentation introduces arrays, type erasure, compute, and `RecordBatch`, but the actual machinery lives in the smaller crates underneath.

That is an important reading hint: if you want to understand *behavior*, read `arrow-array`, `arrow-buffer`, `arrow-data`, and `arrow-schema`. If you want to understand *ergonomics*, read `arrow/src/lib.rs`.

## Design policy visible in the crate docs

Several implementation policies are visible immediately:

1. Arrow in Rust prefers static typing when possible.
   Generic functions over `PrimitiveArray<T>` specialize cleanly.

2. Type erasure is still first-class.
   `ArrayRef = Arc<dyn Array>` makes heterogeneous columns and dynamic dispatch practical.

3. Zero-copy is treated as a default path, not a special optimization.
   `Buffer` can be built from `Vec`, `bytes::Bytes`, foreign allocations, or mmap-backed memory.

4. Rust idioms win over cross-language symmetry when the cost is lower.
   `arrow-array` explicitly does not provide a `ChunkedArray` abstraction and instead recommends `Vec<ArrayRef>`, iterators, or streams.

That last point is worth noticing. The Rust implementation does not mechanically mirror the Python or C++ surface area. It keeps the Arrow memory model while adopting Rust-native composition patterns.

## What to read next

Before reading about `DataType`, `Field`, or `RecordBatch`, understand the low-level buffer contracts. Every higher-level Arrow type eventually reduces to:

- aligned bytes,
- optional validity bits,
- offsets or fixed-width slots,
- child arrays for nesting.

## Source markers

- Workspace members and crate boundaries:
  [`Cargo.toml`](../../Cargo.toml)
- Facade crate:
  [`arrow/src/lib.rs`](../../arrow/src/lib.rs)
- Feature surface and dependencies:
  [`arrow/Cargo.toml`](../../arrow/Cargo.toml)
- Typed-array crate overview and explicit `ChunkedArray` policy:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L1),
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L164)
