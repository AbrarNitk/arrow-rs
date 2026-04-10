# 06. Write Path

Reading Parquet teaches you how the file is interpreted. Writing Parquet teaches you why the file is shaped this way in the first place.

The key writing question is:

**What facts about the file can only be known after data has already been emitted, and how does the format accommodate that?**

Parquet's writer architecture is the answer.

## Start with the constraint

At the moment a file is opened, the writer may know:

- the schema,
- writer properties,
- intended encodings and compression policies.

But it does **not** yet know:

- the final row count,
- actual compressed sizes,
- page boundaries,
- dictionary sizes,
- row group sizes,
- page index offsets,
- many statistics,
- final footer length.

That is why the writer cannot emit a fully accurate master header at the start of the file.

## The writer's real job

A Parquet writer is not just “dump encoded columns.”

It must:

1. serialize data pages and dictionary pages,
2. measure what actually happened on disk,
3. collect enough metadata to describe those bytes later,
4. patch that metadata together into a final footer directory.

This is why the code is full of close-time callbacks and metadata accumulation.

## Control-flow sketch

```text
SerializedFileWriter
   │
   ├─ write opening magic immediately
   │
   ├─ next_row_group()
   │    │
   │    ▼
   │  SerializedRowGroupWriter
   │    │
   │    ├─ next_column()
   │    │    │
   │    │    ▼
   │    │  ColumnWriterImpl
   │    │    ├─ encode levels
   │    │    ├─ encode values
   │    │    ├─ compress page bodies
   │    │    ├─ emit pages
   │    │    └─ return ColumnCloseResult
   │    │
   │    └─ assemble RowGroupMetaData
   │
   └─ write_metadata()
        ├─ write optional indexes / bloom filters
        ├─ serialize FileMetaData
        └─ append footer length + magic
```

## Step 1: open the file and write the leading magic

`SerializedFileWriter::new` immediately writes the starting magic bytes.

This is one of the few things the writer knows with certainty at byte 0.

It also constructs `SchemaDescriptor` up front, because everything downstream depends on leaf-column ordering and level limits.

## Step 2: open a row group

`next_row_group()` returns a `SerializedRowGroupWriter`.

Why does the writer enforce sequential row-group creation?

Because Parquet is fundamentally a forward-emitted format:

- row group bytes land in file order,
- chunk offsets depend on where earlier bytes ended,
- metadata for a row group is only complete after all its columns are closed.

Allowing arbitrary interleaving would make offset accounting and footer synthesis much harder.

## Step 3: write one leaf column chunk at a time

Inside a row group, columns are written in leaf order.

That is a consequence of the schema flattening we discussed earlier:

- logical nested fields are shredded into primitive leaves,
- each leaf becomes one physical chunk,
- each chunk becomes pages.

`ColumnWriterImpl` is where the abstract schema rules become physical bytes:

- repetition levels are encoded if needed,
- definition levels are encoded if needed,
- non-null values are encoded,
- pages are compressed and emitted,
- page-local statistics are collected,
- optional page index and bloom-filter state is accumulated.

This is the moment where a logical leaf column turns into actual page bytes on disk.

## Why close-time results are so important

When a column closes, the writer finally knows facts it could not have known earlier:

- total compressed size of the chunk,
- actual page offsets,
- actual encodings used,
- chunk statistics,
- dictionary presence,
- optional index payloads.

That is why `ColumnCloseResult` exists. It is effectively the writer saying:

“Now that I have actually emitted the bytes, here is the metadata you can safely record about them.”

## Step 4: close the row group

Closing a row group is another metadata synthesis point.

Now the writer can validate and assemble:

- per-column chunk metadata,
- row-group row count consistency,
- bloom filters for the row group,
- column indexes,
- offset indexes.

This means row groups are not just byte partitions. They are also the level at which chunk metadata becomes coherent enough to report upward.

## Step 5: write metadata only after data and side structures exist

`SerializedFileWriter::write_metadata` is where the file becomes self-describing.

The metadata writer:

- serializes optional column indexes,
- serializes optional offset indexes,
- records their offsets back into chunk metadata,
- serializes `FileMetaData`,
- appends 4-byte footer length,
- appends final magic bytes.

This is the exact reason the footer lives at the end: only here does the writer have all the information needed to describe the file truthfully.

## Why metadata is written after indexes

Because the footer must point to those indexes.

```text
write data pages first
   │
   ├─ learn page offsets
   ├─ learn chunk sizes
   ├─ build indexes
   ├─ write indexes
   ├─ learn index offsets
   └─ finally write footer that points to everything
```

This is the “backpatching without seeking backward” pattern Parquet uses. The file is largely written forward, and the final footer serves as the patch table.

## Why this makes sense operationally

This design gives Parquet several good properties:

- writing can be mostly streaming and forward-only,
- readers still get strong random-access metadata,
- optional structures can be added without redesigning the whole file prefix,
- the writer can accurately report what was *actually* written rather than what was planned.

That last point matters. Compression ratios, page boundaries, and dictionary effectiveness are runtime facts, not compile-time facts.

## How Arrow writing fits into this

`ArrowWriter` adds a higher-level contract:

- convert Arrow schema to Parquet schema,
- generate levels from Arrow nesting,
- buffer batches into row groups,
- preserve Arrow schema hints.

But it still bottoms out in the same low-level lifecycle:

- file writer,
- row-group writer,
- column writer,
- page writer,
- metadata writer.

So if you want to truly understand Parquet output, the low-level writer is the right layer to study first.

## Reimplementation checklist

A compatible writer needs:

1. schema flattening,
2. row-group lifecycle management,
3. per-column page emission,
4. page and chunk statistics collection,
5. optional side-structure emission,
6. final footer assembly after all offsets are known.

This is the minimum architecture implied by the format.

## Source markers

- File writer lifecycle and startup:
  [`parquet/src/file/writer.rs`](../../parquet/src/file/writer.rs#L145)
- Row-group writer creation:
  [`parquet/src/file/writer.rs`](../../parquet/src/file/writer.rs#L176)
- Column writer API and close result:
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs#L63),
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs#L193)
- Metadata writer and footer assembly:
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L52),
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L184)
- Arrow writer façade:
  [`parquet/src/arrow/arrow_writer/mod.rs`](../../parquet/src/arrow/arrow_writer/mod.rs#L177)
