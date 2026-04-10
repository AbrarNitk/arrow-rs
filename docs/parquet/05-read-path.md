# 05. Read Path

This chapter answers a practical systems question:

**Given only file bytes, how does a reader recover logical records or Arrow arrays without ever seeing a row-oriented on-disk layout?**

The short answer is:

1. discover the file from the tail,
2. choose row groups and columns from metadata,
3. turn column chunk byte ranges into page streams,
4. decode page headers, level streams, and values,
5. reconstruct records from those streams.

## Start with the true abstraction boundary: `ChunkReader`

Everything begins with `ChunkReader`.

That trait abstracts the only two things the format truly needs from storage:

- read sequentially from an offset,
- materialize a direct byte range.

This is a deep design choice. The core reader is not tied to:

- local files,
- memory-mapped files,
- object stores,
- in-memory byte slices.

It is tied to *range-addressable bytes*.

## Why the read path starts with metadata, not data

Parquet is not a format where you can safely “just start decoding pages” from byte 0.

You first need the footer to learn:

- schema,
- row group boundaries,
- column chunk boundaries,
- compression and encoding metadata,
- optional page indexes,
- statistics for pruning.

So the read path is metadata-first by design.

## Control-flow sketch

```text
ChunkReader
   │
   ▼
ParquetMetaDataReader
   │
   ▼
SerializedFileReader
   │
   ▼
SerializedRowGroupReader
   │
   ▼
SerializedPageReader
   │
   ├─ page header decode
   ├─ decompression
   └─ Page
       │
       ▼
ColumnReaderImpl
   ├─ repetition levels
   ├─ definition levels
   └─ values
       │
       ▼
records / arrays
```

The important observation is that the file reader is mostly a coordinator. The actual format semantics become concrete only once page headers and level/value streams are decoded.

## Step 1: parse metadata from the tail

`SerializedFileReader::new` delegates to `ParquetMetaDataReader`.

This is where the reader learns the file directory:

- number of row groups,
- schema descriptor,
- chunk offsets and sizes,
- optional page index availability.

If page indexes are requested, the metadata reader may need extra reads because Parquet metadata is not necessarily one contiguous footer blob.

That is why `ParquetMetaDataReader` has `NeedMoreData` and prefetch-related logic: tail-oriented readers often work by iteratively fetching just enough of the file suffix to complete metadata discovery.

## Step 2: choose a row group

`FileReader::get_row_group` returns a `SerializedRowGroupReader`.

This step matters because Parquet's main horizontal partition is the row group, not the page.

Why row groups?

- they are large enough to amortize metadata,
- small enough for pruning and parallelism,
- they keep all leaf columns for the same row interval aligned logically.

So row-group selection is the reader's first major pruning decision.

## Step 3: turn metadata into column-chunk reads

For a specific leaf column, the row-group reader constructs a `SerializedPageReader`.

That page reader starts from `ColumnChunkMetaData::byte_range()`. This is the moment when abstract footer offsets become actual storage requests.

There are two important modes:

### Without `OffsetIndex`

```text
chunk start -> read PageHeader -> read page body -> repeat until chunk end
```

The reader must walk the chunk sequentially and discover page boundaries from the headers themselves.

### With `OffsetIndex`

```text
PageLocation[0] -> jump directly to page 0
PageLocation[1] -> jump directly to page 1
PageLocation[2] -> jump directly to page 2
```

Now the reader can treat page locations as a second-level directory and skip linear header walking in some cases.

This is why page indexes matter operationally: they turn pages into seekable units instead of only sequentially discoverable units.

## Step 4: decode page headers correctly

The page reader has to do more than just parse a thrift struct.

It must determine:

- page type,
- compressed and uncompressed sizes,
- whether a dictionary page is present,
- how to split V2 level bytes from value bytes,
- when a page boundary is also a record boundary.

This is one of the most format-sensitive parts of the implementation. A bug here will usually surface as downstream corruption in levels, values, or row reconstruction rather than as a clean local error.

## Step 5: decompress, but only according to page semantics

Page bodies are not just “compressed blobs.”

For V1, the body is conceptually one encoded payload that contains levels and values.
For V2, the body is semantically segmented, and the reader must honor the explicit lengths and `is_compressed` meaning.

This is why page decoding logic cannot be reduced to “read header, decompress body, done.” The page version changes the interpretation of the body.

## Step 6: `ColumnReaderImpl` reconstructs meaning from three streams

At the column-reader layer, the reader has:

- repetition levels,
- definition levels,
- encoded non-null values.

Now it must answer record-oriented questions:

- how many logical records were completed?
- how many levels were consumed?
- how many actual values were produced?

These are different counters.

That distinction is the core of nested Parquet decoding. A repeated nullable column may consume many structural entries while yielding fewer materialized values.

## Why page boundaries are tricky

The code around `at_record_boundary` exists because page boundaries and logical record boundaries are not the same concept.

If you are page-skipping or reconstructing records incrementally, you must know whether stopping at a page boundary is semantically safe or whether a logical record continues across pages.

That is one of the more subtle behind-the-scenes issues in Parquet readers.

## What the reader is really doing conceptually

The deepest mental model is this:

```text
footer gives directory
directory gives chunk byte ranges
chunk bytes give page sequence
pages give level/value streams
level/value streams give logical rows
```

Each stage resolves ambiguity left by the previous one.

## Reimplementation checklist

A compatible reader needs:

1. tail-first metadata parsing,
2. row-group and column-chunk selection,
3. page-header parsing and page-body slicing,
4. decompression logic aware of page version,
5. level decoders,
6. value decoders,
7. row/array reconstruction logic.

If any one of those is missing, the format cannot be reconstructed end-to-end.

## Source markers

- Core reader traits:
  [`parquet/src/file/reader.rs`](../../parquet/src/file/reader.rs#L36)
- File reader construction:
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs#L87)
- Row-group selection:
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs#L277)
- Page decode path:
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs#L387)
- Page reader state machine:
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs#L525)
- Metadata reader and `NeedMoreData` behavior:
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L31),
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L222)
- Column reader:
  [`parquet/src/column/reader.rs`](../../parquet/src/column/reader.rs#L107)
