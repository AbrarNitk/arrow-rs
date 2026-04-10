# 7. FFI And Zero-Copy Boundaries

This chapter answers the interop question:

**How can Arrow data cross library or language boundaries without re-encoding values into a second representation first?**

The answer is the Arrow C Data Interface and the FFI support in `arrow-rs`.

## Why Arrow can support low-overhead FFI in the first place

Arrow's layout already has the right properties:

- buffers are explicit,
- children are explicit,
- nullability is explicit,
- slicing is metadata-driven,
- schema is separate from data.

That means exporting an array is largely a matter of exporting:

- pointers,
- lengths,
- child relationships,
- release semantics.

In a row/object-heavy model, far more hidden state would have to be reconstructed or normalized first.

## Why the FFI layer is split into schema-side and data-side structs

`arrow-rs` uses:

- `FFI_ArrowSchema` in `arrow-schema`
- `FFI_ArrowArray` in `arrow-data`

This mirrors a core Arrow separation:

- schema describes meaning,
- array data describes memory.

That split is not cosmetic. It allows:

- schema exchange without body exchange,
- body exchange under an already-known schema,
- composition with systems that separately track meaning and storage.

## Why raw pointers are not enough

The most important implementation detail is the release callback plus `private_data`.

If the FFI structs only stored raw pointers, they would answer:

- where the data is,

but not:

- who owns it,
- when it may be freed,
- how children and dictionaries stay alive,
- how exported metadata strings are reclaimed.

So the real FFI contract is:

```text
pointer graph + ownership protocol
```

not just:

```text
some C structs with fields
```

## Why release callbacks are the critical safety mechanism

When Rust exports Arrow values:

- Rust still owns the underlying buffers or knows who does,
- the FFI structs point into that owned state,
- a release callback knows how to reclaim the private state later.

```text
Rust values
   │
   ▼
FFI_ArrowSchema / FFI_ArrowArray
  pointers
  children
  dictionary
  release fn
  private_data
   │
   ▼
foreign consumer
   │
   ▼
calls release exactly once when done
```

That callback is what turns a dangerous raw-pointer export into a tractable ownership handoff.

## Why `from_raw` is unsafe

Importing from raw FFI structs is unsafe because the caller is asserting:

- the pointers are valid,
- the struct is initialized correctly,
- ownership transfer rules are respected,
- release semantics are coherent.

This is the natural mirror of `ArrayData::new_unchecked`: the system must have an explicit boundary where external bytes/pointers become trusted Arrow state.

## Why FFI is lower-level than IPC

Arrow IPC and FFI are related but different:

- FFI shares a live Arrow layout across a process or library boundary,
- IPC serializes Arrow layout into a framed byte protocol for files/streams.

Both can be zero-copy friendly, but FFI is closer to the raw memory model while IPC adds transport framing, message boundaries, and versioning.

## Why zero-copy mmap examples matter here too

The example `arrow/examples/zero_copy_ipc.rs` is conceptually adjacent to FFI because it shows the same underlying promise:

- if bytes already satisfy Arrow layout rules,
- higher layers can construct arrays mostly by wiring pointers and metadata,
- not by rebuilding values cell by cell.

That is the practical meaning of Arrow's interop story.

## Reimplementation checklist

To build Arrow interop seriously, you need:

1. schema export/import,
2. array export/import,
3. ownership carried by release callbacks,
4. explicit private state to keep pointer graphs alive,
5. unsafe trust boundaries for raw imports.

That is the FFI layer.

## Source markers

- `FFI_ArrowSchema`:
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L77)
- Schema release callback and private data:
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L97),
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L123)
- `FFI_ArrowArray`:
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L39)
- Array release callback and private data:
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L70),
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L119)
- `from_raw` boundaries:
  [`arrow-schema/src/ffi.rs`](../../arrow-schema/src/ffi.rs#L244),
  [`arrow-data/src/ffi.rs`](../../arrow-data/src/ffi.rs#L224)
- Zero-copy IPC example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs#L1)
