# 7. FFI And Zero-Copy Boundaries

Arrow becomes much more useful when one process or library can hand memory to another without re-encoding every value. That is the job of the Arrow C Data Interface and the FFI support in `arrow-rs`.

## Two sides of the FFI boundary

The implementation splits schema and data ownership cleanly:

- `arrow-schema/src/ffi.rs` exports/imports `FFI_ArrowSchema`
- `arrow-data/src/ffi.rs` exports/imports `FFI_ArrowArray`

That mirrors Arrow's conceptual split:

- schema tells consumers how to interpret the buffers,
- array data points at the actual buffers and children.

## Why Arrow FFI works well

The C Data Interface does not require a process to serialize into IPC first. Instead, it passes:

- pointers,
- lengths,
- child pointers,
- release callbacks,
- format strings and metadata.

In other words, it exports the Arrow in-memory contract directly.

## Ownership is carried by release callbacks

The most important implementation detail is not the struct fields themselves. It is the release mechanism.

When Rust exports Arrow data to C:

- Rust allocates or references the underlying storage,
- the exported struct stores pointers to that storage,
- a `release` callback knows how to free or drop the owned private data later.

This is why the FFI structs carry `private_data`. Raw pointers alone are not enough; somebody must own the lifetime graph.

```text
Rust Arrow values
   │
   ▼
FFI_ArrowSchema / FFI_ArrowArray
   pointers + children + dictionary + release fn + private_data
   │
   ▼
Foreign consumer
   │
   ▼
calls release when done
```

## Why this design matches Arrow's buffer model

Arrow arrays are already:

- contiguous,
- sliceable,
- reference-counted or externally owned,
- explicit about child arrays and null buffers.

That means they map naturally onto FFI structs. There is much less hidden state than in object-heavy row models.

## Zero-copy mmap example

The example `arrow/examples/zero_copy_ipc.rs` is worth reading even before the IPC chapter. It shows the practical path:

1. memory-map an IPC file,
2. wrap the mmap region in `bytes::Bytes`,
3. wrap that in `arrow_buffer::Buffer`,
4. let `FileDecoder` produce arrays that reference the mapped memory directly.

This is Arrow's promise in concrete form: if the bytes already match Arrow layout rules, reading can become mostly a metadata and pointer-construction exercise.

## FFI is lower-level than IPC

It helps to distinguish them:

- FFI: share already-live Arrow memory across a process or language boundary.
- IPC: serialize Arrow messages into a byte protocol for streams or files.

Both are zero-copy friendly, but FFI is closer to the raw memory model while IPC adds framing and transport structure.

## Source markers

- Schema-side C Data Interface:
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L22),
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L77)
- Array-side C Data Interface:
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L27),
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L39)
- Zero-copy IPC example with mmap:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs#L1)
