# 1. Why Arrow Exists And Why `arrow-rs` Is Split Into Many Crates

This chapter starts from the design problem, not the APIs.

The key question is:

**What kind of in-memory representation lets analytical systems scan, slice, share, and serialize 
tabular data without turning every operation into row-by-row object decoding?**

Arrow's answer is a columnar memory contract.

## Start with the performance and interoperability problem

Suppose you store a table in the most obvious object-oriented way:

```text
Vec<Row>
Row {
  id: Option<i32>,
  name: Option<String>,
  score: Option<f64>,
}
```

This is easy to understand, but it is bad for many analytical workloads:

- values from one column are physically scattered across rows,
- each row carries object and pointer overhead,
- strings add another layer of indirection,
- nullability becomes per-value branching state,
- slicing and projection often imply copying or rebuilding.

Now ask what a scan kernel wants instead when computing `sum(score)` or filtering `id > 1000`:

- one dense memory region per column,
- predictable element stride,
- nullability stored orthogonally,
- few branches,
- cheap slicing by offset/length,
- no per-row object traversal.

That is Arrow's starting point.

## Arrow is primarily a memory format, not a file format

This distinction is easy to blur because users often first meet Arrow through IPC or Parquet integration.

But Arrow's central promise is:

- define how typed arrays live in memory,
- make that layout stable enough to share across libraries and languages,
- then build IPC, FFI, and file bridges on top.

So the hierarchy of ideas is:

```text
memory layout first
   │
   ├─ compute
   ├─ zero-copy slicing
   ├─ FFI sharing
   └─ IPC serialization
```

If you reverse that mentally and think “Arrow is an IPC format,” the rest of the design is much harder to reason about.

## Why Arrow is columnar

Arrow chooses columns because many analytical operations are column-oriented:

- aggregates scan one column,
- predicates scan one or a few columns,
- projections often discard most columns,
- vectorized kernels prefer same-typed contiguous values.

This does not mean Arrow cannot represent rows. It means rows are reconstructed by position across columns instead of being stored as row objects.

## Why `arrow-rs` is split into many crates

The workspace is not split just for tidiness. The boundaries reflect real design layers:

- `arrow-buffer`: memory allocation, alignment, shared buffers, bitmaps, offsets
- `arrow-data`: generic physical array representation and validation
- `arrow-schema`: logical types, fields, schema metadata
- `arrow-array`: typed arrays, builders, `RecordBatch`, and dynamic `Array` APIs
- `arrow-ipc`: Arrow stream/file serialization
- `arrow-row`: row-oriented derived format for comparison-heavy operators
- `arrow`: user-facing facade crate

That split answers several practical needs:

- low-level buffer code should be reusable without pulling in high-level array APIs,
- schema negotiation should exist without data ownership,
- validation and physical layout logic should be centralized,
- IPC and FFI should build on the memory contract instead of owning it,
- query engines should be able to depend only on the layers they need.

## Why the top-level `arrow` crate is mostly a facade

The top-level `arrow` crate gives users a convenient entry point, but most of the important mechanics live elsewhere.

That is an intentional policy:

- the facade is for ergonomics,
- the lower crates are for architectural truth.

So if your goal is to understand *how Arrow works*, read:

1. `arrow-buffer`
2. `arrow-data`
3. `arrow-schema`
4. `arrow-array`

and only then treat `arrow` as the convenient public surface.

## Why Rust Arrow is not a line-by-line clone of C++ or Python Arrow

The core memory model is shared across implementations, but the surrounding API policy is Rust-specific in places.

One example: `arrow-array` explicitly does **not** provide a built-in `ChunkedArray` abstraction. Instead it recommends:

- `Vec<ArrayRef>`
- iterators
- streams

That decision is not “missing functionality.” It reflects a Rust-native preference:

- reuse standard composition patterns,
- avoid a heavy mid-level abstraction when vectors/iterators/streams already express the same idea,
- make lazy and async pipelines natural.

This is a useful theme for the whole codebase:

- preserve Arrow's core invariants,
- but adopt Rust-native interfaces when that reduces complexity.

## The core mental model

You can think of the Arrow stack as:

```text
logical meaning
   │
   ▼
schema / fields / datatypes
   │
   ▼
generic physical representation
   │
   ▼
buffers / bitmaps / offsets / child arrays
   │
   ▼
aligned memory / foreign memory / mmap / network bytes
```

Every later chapter just fills in one layer of that picture.

## Reimplementation checklist

If you had to build another Arrow implementation, this chapter says you would need to separate:

1. memory/container primitives,
2. physical array layout validation,
3. logical schema,
4. typed user-facing arrays,
5. transport and interop layers.

That separation is the architecture.

## Source markers

- Workspace boundaries:
  [`Cargo.toml`](../../Cargo.toml)
- Facade crate:
  [`arrow/src/lib.rs`](../../arrow/src/lib.rs#L1)
- Feature surface:
  [`arrow/Cargo.toml`](../../arrow/Cargo.toml)
- `ChunkedArray` policy:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs#L164)
