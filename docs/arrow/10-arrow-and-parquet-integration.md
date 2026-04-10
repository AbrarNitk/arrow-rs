# 10. Arrow And Parquet: Memory Format Versus File Format

Arrow and Parquet are often used together, but they solve different problems.

## Different layers of the data stack

```text
Arrow
  in-memory columnar layout
  zero-copy friendly
  execution and interchange oriented

Parquet
  persisted columnar file format
  encoding/compression/page oriented
  storage and transport oriented
```

Arrow is about the shape of data in memory. Parquet is about the shape of data on disk.

That is why `arrow-rs` keeps Parquet integration in the `parquet` crate rather than burying it inside `arrow-array` or `arrow-data`.

## Where the bridge lives

The Arrow-Parquet bridge lives mainly in:

- `parquet/src/arrow/mod.rs`
- `parquet/src/arrow/schema/mod.rs`
- `parquet/src/arrow/arrow_reader`
- `parquet/src/arrow/arrow_writer`

Those modules convert between:

- Arrow logical schema and arrays,
- Parquet logical/physical schema and pages.

## Why the embedded Arrow schema hint exists

The docs in `parquet/src/arrow/mod.rs` explain that Parquet and Arrow do not always have a one-to-one type mapping. For example, one Parquet physical representation may correspond to multiple Arrow array choices.

To preserve round-tripping, the writer stores the original Arrow schema in Parquet metadata under `ARROW:schema`.

Important detail:

- the Arrow schema is encoded using Arrow IPC,
- then base64 encoded,
- then stored in Parquet key-value metadata.

This is a clever layering decision:

- Arrow uses its own schema serialization format,
- Parquet stores it as metadata,
- readers recover richer Arrow semantics when possible.

## Mental model of the conversion boundary

```text
Arrow RecordBatch
   │
   ▼
Arrow -> Parquet schema conversion
   │
   ▼
row groups / column chunks / pages
   │
   ▼
Parquet file

and on read:

Parquet file
   │
   ▼
Parquet schema + optional ARROW:schema hint
   │
   ▼
Arrow schema reconstruction
   │
   ▼
Arrow arrays / RecordBatch
```

## Why this chapter belongs late in the sequence

You need the earlier chapters first because the bridge assumes you already understand:

- Arrow logical types,
- Arrow physical memory layout,
- `RecordBatch`,
- IPC schema encoding.

Without those pieces, `ARROW:schema` looks like an implementation trick. After reading the IPC and schema chapters, it becomes a natural design choice.

## Source markers

- Arrow-Parquet integration overview:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs#L1)
- Embedded schema metadata key:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs#L220)
- Schema conversion and embedded hint handling:
  [`parquet/src/arrow/schema/mod.rs`](../../parquet/src/arrow/schema/mod.rs)
