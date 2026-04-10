# 2. Memory, Buffers, Bitmaps, And Alignment

This chapter answers a machine-level question:

**What memory properties must hold if higher layers are going to reinterpret raw bytes as typed arrays cheaply and safely?**

Arrow's buffer layer is the answer.

## Start with the hardware problem

CPUs are fast when work looks like:

- sequential loads,
- predictable stride,
- low branch entropy,
- minimal pointer chasing,
- high cache reuse,
- layouts the compiler can vectorize.

They are much slower when work looks like:

- follow a pointer to a row object,
- follow another pointer to a string object,
- inspect an enum tag for nullability,
- branch on every value,
- repeat for millions of rows.

Arrow's buffer model exists to keep the hot path in the first category.

## Why buffers instead of per-value objects

The basic Arrow policy is:

- store values contiguously,
- move variability into side structures,
- make nullability orthogonal,
- make slicing metadata-only whenever possible.

That is why the recurring storage shapes are so simple:

```text
fixed-width values:
  [v0][v1][v2][v3]...

validity bitmap:
  bit0 bit1 bit2 bit3 ...

offsets + values:
  offsets: [0][5][5][9]...
  values : [h e l l o r u s t]
```

The entire Arrow system is built on these few shapes.

## `arrow-buffer` is the “legal memory” layer

`arrow-buffer` defines the ways bytes may exist if they are to participate in Arrow:

- `MutableBuffer`: growable writable memory
- `Buffer`: immutable shared sliceable memory
- `BooleanBuffer`: packed bits
- `NullBuffer`: validity semantics on top of bits
- `ScalarBuffer<T>`: typed fixed-width view
- `OffsetBuffer<O>`: validated monotonic offsets

Think of this crate as defining the lawful storage substrate for everything above it.

## Why alignment is an explicit throughput policy

The comments in `arrow-buffer/src/alloc/alignment.rs` are unusually direct: the alignment choices are about spatial and temporal prefetcher behavior and cache-aware allocation, not just ABI correctness.

On `x86_64`, `ALIGNMENT` is `128` bytes.

That tells you a lot about the intended use case:

- Arrow expects scan-heavy workloads,
- it is willing to over-align memory to help those scans,
- the buffer layer is designed with CPU behavior in mind, not just type safety.

This is one of the clearest examples of “policy baked into mechanics” in the repo.

## Why `MutableBuffer` rounds capacity up

`MutableBuffer::with_capacity` rounds requested capacity up to a multiple of 64 bytes.

That is not just an allocation convenience. It protects several later assumptions:

- reallocation frequency stays lower,
- alignment remains predictable,
- buffers produced from builders have shapes compute code expects,
- later conversion into immutable `Buffer` preserves the scan-friendly layout.

```text
requested: 70 bytes
rounded:  128 bytes

cost:
  some extra space

benefit:
  better alignment story
  fewer reallocations
  more stable throughput
```

This is a typical Arrow tradeoff: pay a little extra memory to keep the hot path regular.

## Why `Buffer` stores a pointer, not only an offset

`Buffer` is immutable, shared, and sliceable. The especially important design detail is this comment in the implementation:

- it stores a pointer into the data, not merely a base pointer plus offset,
- specifically to avoid pointer arithmetic patterns that hurt LLVM vectorization.

This is exactly the kind of low-level choice you would miss if you only looked at the API surface.

The design pressure is:

- sliced arrays must be cheap,
- but the representation of a slice should still generate good compiled code.

So the data structure itself is shaped for compiler behavior.

## Why nullability is a bitmap

`NullBuffer` wraps a packed bit buffer where:

- `1` means valid,
- `0` means null.

Why not store `Option<T>` values directly?

- it inflates fixed-width arrays,
- it destroys dense value layout,
- it adds per-value branching and tagging cost,
- it makes interoperability harder because the null representation is no longer orthogonal.

The bitmap design says:

- keep the values buffer dense,
- keep nullability separate,
- let kernels combine validity and value scans explicitly.

That is why null positions may still contain arbitrary but well-formed bytes in the values buffer.

## Why offsets solve variable-width data cleanly

For `Utf8`, `Binary`, `List`, and related types, Arrow uses:

- one offsets buffer,
- one values buffer,
- optional null bitmap.

Example:

```text
logical values = ["hello", "", "rust"]

offsets = [0, 5, 5, 9]
values  = [h e l l o r u s t]

slot 0 -> values[0..5]
slot 1 -> values[5..5]
slot 2 -> values[5..9]
```

This solves several problems at once:

- random access is simple,
- slicing is simple,
- empty values are represented naturally,
- one contiguous values buffer is still possible.

The monotonicity of offsets is what keeps the representation sane and sliceable.

## Why Arrow can accept foreign memory

`Buffer` can be built from:

- `Vec<T>`
- `bytes::Bytes`
- custom allocations
- mmap-backed memory

This is crucial for Arrow's interoperability story. If Arrow required all data to be freshly copied into a special owned container, zero-copy IPC and FFI would collapse into marketing language instead of a real property.

So the low-level buffer layer is deliberately permissive about *where* memory comes from, as long as it can be represented with the required invariants.

## The underlying theme

All of these design choices are variations on one principle:

```text
make physical layout predictable enough that:
  bytes can be interpreted cheaply,
  slices can share storage,
  kernels can vectorize,
  foreign memory can participate
```

That is the reason this chapter belongs before `ArrayData` or `RecordBatch`.

## Reimplementation checklist

To build another Arrow implementation, you would need:

1. aligned growable buffers,
2. shared immutable sliceable buffers,
3. packed bitmaps for booleans/nulls,
4. validated monotonic offsets,
5. a memory ownership model that can wrap foreign allocations.

This is the true substrate of the system.

## Source markers

- Alignment policy:
  [`arrow-buffer/src/alloc/alignment.rs`](../../arrow-buffer/src/alloc/alignment.rs#L17)
- `MutableBuffer` guarantees and growth behavior:
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs#L31),
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs#L113),
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs#L246)
- Rounding helper:
  [`arrow-buffer/src/util/bit_util.rs`](../../arrow-buffer/src/util/bit_util.rs#L24)
- `Buffer` and pointer-layout choice:
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs#L27),
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs#L71)
- `NullBuffer`:
  [`arrow-buffer/src/buffer/null.rs`](../../arrow-buffer/src/buffer/null.rs#L34)
- `OffsetBuffer`:
  [`arrow-buffer/src/buffer/offset.rs`](../../arrow-buffer/src/buffer/offset.rs#L59)
