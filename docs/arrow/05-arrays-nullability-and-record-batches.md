# 5. Typed Arrays, Nullability, And `RecordBatch`

This chapter answers the user-facing question:

**How do generic physical array layouts become ergonomic typed arrays and tabular units without losing the guarantees the lower layers established?**

The answer lives in `arrow-array`.

## Why typed arrays exist if `ArrayData` already exists

`ArrayData` is the universal physical contract, but it is not ergonomic enough for everyday work.

Users and kernels need:

- typed accessors,
- constructors and builders,
- type-erased dynamic dispatch when schemas are heterogeneous,
- a batch-level tabular unit.

So `arrow-array` adds those, but it does not replace the lower contract. It is built on top of it.

## Why `Array` is an `unsafe trait`

This is one of the most important policy signals in the repo.

The trait is `unsafe` because implementations must uphold invariants the rest of the system assumes without re-checking:

- `len` must match accessible memory,
- `data_type` must truthfully describe layout,
- slicing must preserve invariants,
- `to_data()` must return coherent `ArrayData`.

Why this matters:

- kernels may use lengths and types to index memory directly,
- downcasting logic assumes type/layout agreement,
- invalid implementations can lead to panics or undefined behavior.

The docs explicitly warn that third-party implementations are probably not safe to get right in general. That is not gatekeeping; it is an honest reflection of how much the rest of the library assumes from `Array`.

## Why both typed arrays and type-erased arrays are necessary

Arrow Rust uses two complementary access styles:

### Concrete typed arrays

Examples:

- `Int32Array`
- `StringArray`
- `ListArray`

Benefits:

- specialized methods,
- native-value access,
- monomorphized generic kernels,
- less runtime casting.

### Type-erased arrays

Examples:

- `&dyn Array`
- `ArrayRef = Arc<dyn Array>`

Benefits:

- heterogeneous columns in one batch,
- generic pipelines,
- schema-driven processing,
- transport and execution boundaries.

This is not duplication. It is the natural split between performance-oriented specialization and schema-driven dynamism.

## Why nullability is separate from values even at the typed-array layer

For fixed-width arrays like `Int32Array`, the values buffer remains dense even when logical positions are null.

```text
values:   [10][99][30][40]
validity:  1   0   1   1

logical:
  0 -> 10
  1 -> null
  2 -> 30
  3 -> 40
```

That means:

- null positions do not remove slots from the values buffer,
- `values()` may contain arbitrary but well-formed bytes for null positions,
- the bitmap remains the source of truth.

This is the same orthogonality principle we saw at the buffer layer, now expressed through typed APIs.

## Why primitive arrays are the ideal Arrow case

`PrimitiveArray<T>` is the simplest Arrow success case:

- optional validity bitmap,
- dense native values.

That is why numeric kernels are often the easiest place to see Arrow's performance model clearly:

- maybe scan a bitmap,
- scan a dense slice,
- perform the operation.

Minimal indirection. Minimal ambiguity.

## Why variable-width typed arrays still reduce to the same few shapes

`StringArray` / `BinaryArray` add offsets and a values buffer, but the underlying philosophy does not change.

You still have:

- optional validity,
- one structural side buffer,
- one dense payload buffer.

That regularity is why generic code can still reason about them and why slicing remains cheap.

## Why `RecordBatch` is the unit of tabular work

A `RecordBatch` is:

- one schema,
- multiple columns,
- all columns same logical row count.

It is the point where Arrow says:

“These arrays line up positionally and therefore form one tabular slice.”

```text
RecordBatch
  schema: [id, name, score]
  columns:
    col0 len = N
    col1 len = N
    col2 len = N

row i = (col0[i], col1[i], col2[i])
```

This is why `RecordBatch` is the common unit for:

- IPC,
- Parquet bridges,
- query engines,
- streaming pipelines.

It is large enough to amortize overhead, small enough to stream, and neutral about compute strategy.

## Why Rust Arrow deliberately avoids `ChunkedArray`

The crate docs explicitly recommend:

- `Vec<ArrayRef>`
- iterators
- streams

instead of a built-in `ChunkedArray`.

This is a policy decision that reflects Rust idioms:

- reuse standard collection and iterator abstractions,
- keep async and lazy execution natural,
- avoid a large extra abstraction layer when composition already works.

This is another example of the project preserving Arrow's core model without copying every API convention from other language implementations.

## Why `RecordBatchReader` and `RecordBatchWriter` matter

These traits define the tabular IO contract:

- readers yield batches under a stable schema,
- writers consume batches and finalize on `close`.

That is what lets IPC, Parquet, and execution engines all plug into one common shape without agreeing on a row representation.

## Reimplementation checklist

To build another Arrow implementation, you need:

1. typed arrays on top of the physical contract,
2. a dynamic type-erased array boundary,
3. explicit null semantics,
4. a batch-level tabular unit,
5. a streaming/batch IO contract.

That is the usability layer of Arrow.

## Source markers

- `Array` safety boundary:
  [`arrow-array/src/array/mod.rs`](../../arrow-array/src/array/mod.rs#L82)
- Crate overview and `ChunkedArray` policy:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L1),
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L164)
- Variable-width arrays:
  [`arrow-array/src/array/byte_array.rs`](../../arrow-array/src/array/byte_array.rs#L87)
- `RecordBatch`, reader, writer:
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L30),
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L45),
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L224)
