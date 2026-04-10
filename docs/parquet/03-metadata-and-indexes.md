# 03. Metadata And Indexes

Once you understand the file tail and the schema flattening story, the next question is:

**What information must the footer preserve so a reader can plan IO and decode data without scanning the whole file body first?**

Parquet metadata is the answer to that question.

## Metadata is a directory, not decoration

It is tempting to think of metadata as “extra information about the file.”

That is the wrong mental model.

For Parquet, metadata is the *directory service* that makes the file usable:

- it tells the reader what the schema is,
- where row groups are,
- where column chunks begin,
- what encodings and compression were used,
- what statistics exist,
- and where optional side structures such as page indexes live.

Without the footer, the bytes in the body are not meaningfully navigable except by brute-force scanning.

## The metadata hierarchy mirrors the storage hierarchy

`arrow-rs` models Parquet metadata with:

- `ParquetMetaData`
- `FileMetaData`
- `RowGroupMetaData`
- `ColumnChunkMetaData`

That mirrors the physical layout exactly:

```text
ParquetMetaData
├─ FileMetaData
│  ├─ schema descriptor
│  ├─ total row count
│  ├─ created_by
│  └─ key/value metadata
├─ RowGroupMetaData[0]
│  ├─ row count
│  ├─ byte footprint
│  └─ ColumnChunkMetaData[*]
├─ RowGroupMetaData[1]
│  └─ ColumnChunkMetaData[*]
├─ ColumnIndex?   optional page-level stats
└─ OffsetIndex?   optional page-level byte locations
```

This is not accidental. It means planning can happen at the same granularity as storage:

- file-wide decisions from `FileMetaData`,
- row-group pruning from `RowGroupMetaData`,
- column and page planning from `ColumnChunkMetaData` plus page indexes.

## What `FileMetaData` is really for

`FileMetaData` answers file-wide questions:

- what schema should the body be interpreted with?
- how many rows are in the whole file?
- which writer created this file?
- what file-level key/value metadata exists?

This is enough for a reader to understand the *shape* of the data before touching any actual page bytes.

But `FileMetaData` is intentionally not enough to execute a scan. For that you need row groups and column chunks.

## What `RowGroupMetaData` is really for

`RowGroupMetaData` is the first real pruning unit.

It tells you:

- how many rows are in this row group,
- how large it is,
- which column chunks belong to it,
- and where they sit logically in the file hierarchy.

Why this matters:

- query engines often skip whole row groups using row-group-level stats,
- parallel readers commonly assign work per row group,
- row groups preserve the alignment of all leaf columns for the same row interval.

So row groups are where “logical row partitioning” meets “columnar physical layout.”

## What `ColumnChunkMetaData` must preserve

This is the most operational metadata object in the format.

It answers:

- what leaf column is this?
- what physical type is stored here?
- how many values exist?
- which encodings were used?
- what codec compressed the chunk?
- where do the data pages begin?
- is there a dictionary page?
- what are the chunk-level statistics?
- are there bloom filters?
- are there page indexes?

This is the object a real scan planner stares at when deciding what bytes to fetch.

## Why there are both chunk statistics and page indexes

These solve different pruning problems.

### Chunk-level metadata

Good for:

- deciding whether to skip an entire row group / column chunk,
- lightweight planning with low metadata overhead.

Bad for:

- fine-grained skipping inside a large chunk.

### Page-level indexes

Good for:

- skipping only some pages,
- planning precise reads,
- supporting more selective scans.

Bad for:

- larger metadata footprint,
- more offset bookkeeping,
- more complicated read path.

So Parquet gives you a hierarchy of pruning granularity instead of forcing one global choice.

## `ColumnIndex` and `OffsetIndex` are a paired design

These two optional structures are best understood together.

### `ColumnIndex`

Tells you page-level content facts such as:

- min value,
- max value,
- null-page information,
- boundary ordering information,
- and in `arrow-rs`, even definition/repetition histograms when available.

This answers:

**Should I read this page at all?**

### `OffsetIndex`

Tells you page-level location facts such as:

- where each data page lives,
- how large it is,
- and how pages line up physically.

This answers:

**If I do want this page, where are its bytes?**

That pairing is elegant:

- `ColumnIndex` says *whether* a page is interesting,
- `OffsetIndex` says *where* the interesting page is.

## Why page indexes are not embedded in the main footer blob

This is a subtle but important design choice.

Page indexes are logically part of metadata, but physically they may live outside the contiguous thrift `FileMetaData` bytes.

Why?

- they are optional,
- they may be large,
- they may be written late,
- the writer wants to record their offsets rather than force one giant monolithic footer layout.

So the footer stores references to page indexes instead of necessarily containing them inline.

```text
... file body ...
├─ bloom filter bytes?          referenced by bloom_filter_offset
├─ column index bytes?          referenced by column_index_offset
├─ offset index bytes?          referenced by offset_index_offset
├─ thrift FileMetaData          records the offsets above
├─ metadata length
└─ PAR1
```

This is why the metadata reader sometimes needs multiple reads even after it has already decoded the footer proper.

## Why `NeedMoreData` exists

`ParquetMetaDataReader` supports partial suffix reading. That only makes sense because:

- the last 8 bytes tell you footer length,
- the footer may tell you that more metadata lives earlier in the file,
- you may have fetched only a short suffix initially.

`NeedMoreData` is therefore not just an implementation convenience. It is a natural consequence of Parquet's tail-first discovery model plus non-contiguous metadata.

## Why metadata writing requires backpatching

The metadata writer first serializes optional page indexes, then records their offsets back into `ColumnChunkMetaData`, then serializes `FileMetaData`.

This order exists because the chunk metadata cannot honestly contain index offsets until those index bytes have been written somewhere.

That is the same basic pattern we saw at the whole-file level:

- write bytes first,
- learn true offsets,
- encode those offsets into later metadata.

## A useful way to think about the footer

The footer is not “the schema plus some stats.”

It is a multi-level navigation structure:

```text
footer
  -> file-wide facts
  -> row-group directory
  -> chunk directory
  -> optional page directories
```

That perspective makes the reader architecture much easier to understand.

## Reimplementation checklist

A serious implementation must provide:

1. file-wide metadata objects,
2. row-group metadata,
3. chunk metadata with offsets and statistics,
4. optional page index parsing and writing,
5. logic for non-contiguous metadata reads,
6. offset patching during write.

If you skip any of these, your format understanding is still incomplete.

## Source markers

- Metadata overview and type layering:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L17)
- `ParquetMetaData`:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L173)
- `FileMetaData`:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L490)
- `RowGroupMetaData`:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L633)
- `ColumnChunkMetaData` and byte-range helpers:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L811),
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L1043)
- Metadata reader and page-index policies:
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L61),
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L84),
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L335)
- Metadata writer patching index offsets:
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L66),
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L99),
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L184)
- Page index types:
  [`parquet/src/file/page_index/column_index.rs`](../../parquet/src/file/page_index/column_index.rs),
  [`parquet/src/file/page_index/offset_index.rs`](../../parquet/src/file/page_index/offset_index.rs)
