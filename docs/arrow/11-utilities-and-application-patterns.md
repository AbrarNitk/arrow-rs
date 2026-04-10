# 11. Utilities, Workflows, And How To Apply Arrow

This chapter answers the practical question:

**Given all the lower-level theory, which Arrow layer should you actually reach for in a real system, and why?**

The goal here is not just API recommendation. It is to connect use cases back to the design logic from earlier chapters.

If you have not yet read the synthesis chapter, read [12-worked-example-from-arrays-to-ipc.md](./12-worked-example-from-arrays-to-ipc.md) before treating these workflows as “just choose the right crate.” The worked example makes the transitions between buffers, arrays, `RecordBatch`, and IPC concrete.

## Workflow 1: Build typed arrays directly

Use typed array constructors or builders when:

- your data already lives in process memory,
- you want direct control over nulls and offsets,
- you are writing compute kernels, tests, or low-overhead ingestion code.

Why this makes sense:

- you stay close to the real memory model,
- there is minimal abstraction overhead,
- you can reason directly about buffers and nullability.

## Workflow 2: Build `ArrayData` when crossing low-level boundaries

Use `ArrayData` / `ArrayDataBuilder` when:

- importing foreign memory,
- writing a custom reader,
- reconstructing arrays from raw buffers,
- validating physical layouts explicitly,
- implementing a transport or FFI bridge.

Why this makes sense:

- `ArrayData` is the invariant boundary,
- it is lower level than typed arrays but still structured enough to validate,
- it is the right place to turn “some bytes” into “trusted Arrow array.”

## Workflow 3: Use `RecordBatch` as the pipeline unit

Use `RecordBatch` when:

- passing tabular data between operators,
- integrating with query engines,
- writing IPC or Parquet,
- building batch-oriented dataflows.

Why this makes sense:

- it preserves schema + aligned column sets,
- it is neutral about execution strategy,
- it is the common contract shared by many Arrow-adjacent systems.

## Workflow 4: Use IPC for transport and cache artifacts

Use `arrow-ipc` when:

- sending Arrow data over the network,
- caching Arrow-native artifacts on disk,
- preserving buffer layout across process boundaries,
- enabling zero-copy-friendly reopen paths such as mmap.

Why this makes sense:

- IPC is closest to the in-memory model,
- reconstruction can often be cheap,
- you keep Arrow semantics without translating into a different data model.

## Workflow 5: Use FFI for language or library boundaries

Use the C Data Interface when:

- another component already speaks Arrow C ABI,
- you want to avoid a full serialization step,
- buffers and schema can stay alive across the boundary.

Why this makes sense:

- FFI is lower-overhead than IPC when both sides can share live memory conventions,
- Arrow's explicit buffer model maps naturally to this interface.

## Workflow 6: Use `arrow-row` for comparison-heavy execution

Use `arrow-row` when:

- multi-column comparison dominates runtime,
- you need normalized sort keys,
- grouping or window partition comparison is hot,
- lexicographic ordering over tuples matters more than pure scan throughput.

Why this makes sense:

- columnar is not optimal for every operator,
- the row format is specifically designed to make tuple comparison cheap,
- you are deriving an execution shape, not replacing Arrow storage.

## Workflow 7: Use Parquet when persistence matters more than staying purely in-memory

Use Parquet when:

- data must be stored efficiently on disk,
- compression and page-level encoding matter,
- row-group and statistics-based pruning matter,
- Arrow is your in-memory representation but not your archival format.

Why this makes sense:

- Arrow is the in-memory contract,
- Parquet is the persisted columnar file format,
- the bridge between them is explicit rather than pretending they are the same thing.

## Example repository paths worth reading

- zero-copy IPC example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs)
- Parquet Arrow bridge examples:
  [`parquet/examples`](../../parquet/examples)

## A productive code-reading path

If your goal is to become productive in `arrow-rs`, read in this order:

1. `arrow-buffer`
2. `arrow-data`
3. `arrow-schema`
4. `arrow-array`
5. `arrow-ipc`
6. `arrow-row`
7. `parquet/src/arrow`

That is the dependency and concept flow of the repo. It is the shortest path from:

```text
"why is this buffer shaped like this?"
```

to:

```text
"how does this become a RecordBatch, an IPC file, or a Parquet file?"
```

## Source markers

- High-level facade:
  [`arrow/src/lib.rs`](../../arrow/src/lib.rs)
- Typed arrays:
  [`arrow-array/src/lib.rs`](../../arrow-array/src/lib.rs)
- `RecordBatch`:
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs)
- IPC example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs)
- Row conversion:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs)
- Parquet bridge:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs)
