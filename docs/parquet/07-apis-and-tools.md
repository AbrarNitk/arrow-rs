# 07. APIs And Tools

This chapter closes the loop by mapping practical APIs and small use cases back to the low-level material from the previous chapters.

If you have not yet read the worked synthesis chapter, read [08-worked-example-from-rows-to-pages.md](./08-worked-example-from-rows-to-pages.md) before treating these APIs as “just builders and helpers.” The high-level APIs make much more sense once you can mentally trace one nested field all the way through levels, pages, and footer metadata.

## Arrow read APIs

`ArrowReaderBuilder` is the central synchronous Arrow read builder.

Its main controls are:

- `with_batch_size`
- `with_row_groups`
- `with_projection`
- `with_row_selection`
- `with_row_filter`

Conceptually:

- metadata decides what can be skipped,
- row groups narrow the scan,
- projection narrows leaf columns,
- row selection narrows page/range work,
- Arrow array readers reconstruct typed arrays from the page/level stream.

### Good use case: selective analytical scan

Use `ParquetRecordBatchReaderBuilder` with:

- a projection mask,
- a selected row-group list,
- optional row selection or filter,
- a tuned batch size.

This is the standard path for analytical reading from local or buffered data.

## Arrow read stack diagram

```text
ParquetRecordBatchReaderBuilder
   ├─ metadata loading
   ├─ projection / row-group selection
   ├─ page readers
   ├─ array readers
   └─ RecordBatch output
```

## Arrow write APIs

`ArrowWriter` is the standard high-level write API.

Good use cases:

- write one or more `RecordBatch` values into a single file,
- flush early when row-group memory usage grows,
- preserve Arrow schema hints for better round-tripping,
- apply standard writer properties without dropping down to manual column writers.

The explicit `flush()` call is worth remembering: it lets you cut a row group early for memory reasons, even though smaller row groups may increase metadata overhead.

## Arrow write stack diagram

```text
RecordBatch
   │
   └─ ArrowWriter
        ├─ Arrow schema -> Parquet schema conversion
        ├─ row-group buffering
        ├─ column writers
        └─ SerializedFileWriter
             └─ Parquet file bytes
```

## Metadata-centric workflows

The metadata APIs are useful by themselves when:

- you want to inspect a file without scanning data pages,
- you want to cache metadata in a catalog or sidecar file,
- you want to prune files or row groups before opening full scans.

The `external_metadata.rs` example is the best compact demonstration of this pattern in the repository.

## CLI and example utilities in this repository

### `parquet-layout`

Use this when you want a physical-layout view:

- row groups,
- column chunks,
- page offsets,
- page header sizes,
- compressed and uncompressed page sizes,
- presence of indexes and bloom filters.

This binary is especially useful after reading Chapters 1 through 4 because it prints exactly the structures those chapters describe.

### `parquet-schema`

Use this when you want:

- the file schema,
- or verbose file metadata for inspection and debugging.

### `parquet-read`

Use this for a simple whole-file record read via the record API. It is not a projection/predicate-optimized tool, but it is useful for understanding the end-to-end row iteration path.

### Example files worth reading

- `read_parquet.rs`: minimal Arrow read path
- `write_parquet.rs`: Arrow write path with bloom-filter configuration
- `external_metadata.rs`: separate metadata loading and reuse
- `read_with_rowgroup.rs`: custom row-group backed Arrow reading

## Small technical scenarios

### Scenario 1: “I want to understand the exact file layout”

Read:

1. [01-file-anatomy.md](./01-file-anatomy.md)
2. [03-metadata-and-indexes.md](./03-metadata-and-indexes.md)
3. [04-pages-and-encodings.md](./04-pages-and-encodings.md)

Then run or inspect `parquet-layout`.

### Scenario 2: “I want to debug a nested column decode bug”

Read:

1. [02-schema-and-nesting.md](./02-schema-and-nesting.md)
2. [04-pages-and-encodings.md](./04-pages-and-encodings.md)
3. [05-read-path.md](./05-read-path.md)
4. [08-worked-example-from-rows-to-pages.md](./08-worked-example-from-rows-to-pages.md)

Pay special attention to `ColumnDescriptor`, level handling, and `ColumnReaderImpl::read_records`.

### Scenario 3: “I want to tune writer layout for query performance”

Read:

1. [06-write-path.md](./06-write-path.md)
2. [08-worked-example-from-rows-to-pages.md](./08-worked-example-from-rows-to-pages.md)
3. `WriterProperties`
4. `write_parquet.rs`

Focus on:

- row-group sizing,
- page sizing,
- statistics level,
- bloom-filter position,
- compression and encoding choices.

### Scenario 4: “I want to reuse metadata separately from remote file I/O”

Read:

1. [03-metadata-and-indexes.md](./03-metadata-and-indexes.md)
2. [05-read-path.md](./05-read-path.md)
3. `external_metadata.rs`

## Source markers

- Arrow schema hint:
  [`parquet/src/arrow/mod.rs:17-52`](../../parquet/src/arrow/mod.rs#L17-L52),
  [`parquet/src/arrow/mod.rs:220`](../../parquet/src/arrow/mod.rs#L220)
- Arrow read builder:
  [`parquet/src/arrow/arrow_reader/mod.rs:107-252`](../../parquet/src/arrow/arrow_reader/mod.rs#L107-L252)
- Arrow reader options and metadata:
  [`parquet/src/arrow/arrow_reader/mod.rs:450-630`](../../parquet/src/arrow/arrow_reader/mod.rs#L450-L630),
  [`parquet/src/arrow/arrow_reader/mod.rs:851-1064`](../../parquet/src/arrow/arrow_reader/mod.rs#L851-L1064)
- Arrow writer:
  [`parquet/src/arrow/arrow_writer/mod.rs:177-289`](../../parquet/src/arrow/arrow_writer/mod.rs#L177-L289),
  [`parquet/src/arrow/arrow_writer/mod.rs:338-439`](../../parquet/src/arrow/arrow_writer/mod.rs#L338-L439)
- Minimal read example:
  [`parquet/examples/read_parquet.rs`](../../parquet/examples/read_parquet.rs)
- Write example with bloom filters:
  [`parquet/examples/write_parquet.rs`](../../parquet/examples/write_parquet.rs)
- External metadata example:
  [`parquet/examples/external_metadata.rs`](../../parquet/examples/external_metadata.rs)
- In-memory row-group example:
  [`parquet/examples/read_with_rowgroup.rs`](../../parquet/examples/read_with_rowgroup.rs)
- CLI tools:
  [`parquet/src/bin/parquet-layout.rs`](../../parquet/src/bin/parquet-layout.rs),
  [`parquet/src/bin/parquet-schema.rs`](../../parquet/src/bin/parquet-schema.rs),
  [`parquet/src/bin/parquet-read.rs`](../../parquet/src/bin/parquet-read.rs)
