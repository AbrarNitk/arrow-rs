# 2. Memory, Buffers, Bitmaps, And Alignment

Arrow performance starts before Arrow types appear. It starts with the assumption that scanning contiguous memory is cheaper than chasing pointers through row objects.

## Why columnar memory matters at CPU level

When a kernel reads one column of `Int32`, it wants:

- one tight values buffer,
- predictable stride,
- minimal branches,
- prefetch-friendly access,
- no object headers between values.

That gives the CPU a chance to use caches, prefetchers, vectorized loads, and branch prediction well.

The Arrow layout is therefore built around a few primitive storage shapes:

```text
Fixed-width values:
  [v0][v1][v2][v3]...

Validity bitmap:
  bit0 bit1 bit2 bit3 ...

Offsets + values:
  offsets: [0][5][5][9]...
  values : [h e l l o a b c d ...]
```

## `arrow-buffer`: the memory substrate

`arrow-buffer` provides the storage types used by the rest of the stack:

- `MutableBuffer`: growable, aligned, writable memory.
- `Buffer`: immutable, shared, sliceable memory.
- `BooleanBuffer`: packed bits.
- `NullBuffer`: validity bitmap with Arrow null semantics.
- `ScalarBuffer<T>`: typed fixed-width view over a `Buffer`.
- `OffsetBuffer<O>`: validated monotonic offsets for variable-sized arrays.

Think of this crate as "the legal ways to hold Arrow bytes."

## Cache-aware alignment is an explicit policy

The code in `arrow-buffer/src/alloc/alignment.rs` is unusually revealing. It does not just say "align for correctness." It says alignment is chosen with spatial and temporal prefetcher behavior in mind.

On `x86_64`, `ALIGNMENT` is `128` bytes. The comments cite Intel guidance and explain that the allocation policy is chosen to match cache and prefetch patterns, not merely ABI minimums.

That gives you the intended design philosophy:

- alignment is a throughput decision,
- layout is chosen for scan-heavy analytics,
- the library is willing to over-align if that helps predictable access.

## `MutableBuffer`: build aligned data first

`MutableBuffer` is the write-time workhorse. It rounds capacities up to multiples of 64 bytes via `round_upto_multiple_of_64`.

That means when builders append values, they are not just pushing bytes into a `Vec<u8>` with arbitrary growth. They are growing storage in a way that preserves the alignment story expected by later readers and compute kernels.

```text
Requested capacity:  70 bytes
Rounded capacity:   128 bytes

Why?
  less frequent reallocations
  predictable alignment
  more cache-friendly scans later
```

## `Buffer`: immutable, shared, sliceable, zero-copy

`Buffer` is the read-time and sharing-time abstraction. Three details matter:

1. it is reference-counted,
2. it can be sliced without copying,
3. it stores a pointer directly, not just a base pointer plus offset.

That third detail is subtle and important. The code comments state that storing a pointer instead of only an offset avoids pointer arithmetic patterns that cause LLVM to miss vectorization opportunities. This is exactly the kind of low-level implementation decision the Arrow stack depends on: the data structure is shaped for compiler optimization, not just for API neatness.

## Validity is packed into bits, not bytes

Nullability is represented by `NullBuffer`, which wraps a `BooleanBuffer`. Each slot uses one bit:

```text
Index:    0 1 2 3 4 5 6 7
Valid?:   1 0 1 1 0 1 1 1
Bitmap:   11101101  (bit order simplified for explanation)
```

Why pack bits?

- nullability is orthogonal metadata,
- one bit per value is far cheaper than one byte or one enum tag,
- validity scans can be fused with value scans.

The important semantic rule is:

- `true` bit => valid value
- `false` bit => null

This lets fixed-width arrays keep dense value buffers even when some logical positions are null.

## Offsets are Arrow's answer to variable-width values

`OffsetBuffer<O>` stores monotonically increasing offsets. For `Utf8`, `Binary`, `List`, and related types, this is how Arrow turns a "sequence of variable-length things" into two contiguous buffers:

```text
logical values = ["hello", "", "rust"]

offsets = [0, 5, 5, 9]
values  = [h e l l o r u s t]

slot 0 -> values[0..5]
slot 1 -> values[5..5]
slot 2 -> values[5..9]
```

The monotonic constraint is what keeps slicing and random access simple.

## Why Arrow prefers buffers over per-value objects

If strings were stored as `Vec<Option<String>>`, each logical value would drag in:

- allocator overhead,
- pointer indirection,
- per-object metadata,
- reduced cache locality.

Arrow instead stores:

- one bitmap,
- one offsets buffer,
- one values buffer.

That is the recurring Arrow pattern: move variability into side structures, keep the main storage contiguous.

## Source markers

- Alignment policy:
  [`arrow-buffer/src/alloc/alignment.rs`](../../arrow-buffer/src/alloc/alignment.rs#L17),
  [`arrow-buffer/src/alloc/alignment.rs`](../../arrow-buffer/src/alloc/alignment.rs#L38)
- Rounding capacities to cache-friendly multiples:
  [`arrow-buffer/src/util/bit_util.rs`](../../arrow-buffer/src/util/bit_util.rs#L24),
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs#L99),
  [`arrow-buffer/src/buffer/mutable.rs`](../../arrow-buffer/src/buffer/mutable.rs#L128)
- Shared immutable buffers and pointer choice:
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs#L71)
- Validity bitmap:
  [`arrow-buffer/src/buffer/null.rs`](../../arrow-buffer/src/buffer/null.rs#L34)
- Variable-width offsets:
  [`arrow-buffer/src/buffer/offset.rs`](../../arrow-buffer/src/buffer/offset.rs#L59)
