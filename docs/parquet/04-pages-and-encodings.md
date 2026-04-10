# 04. Pages And Encodings

This chapter is the byte-level heart of Parquet.

After the footer tells you where a column chunk lives, the next problem is:

**How do we split one leaf column into independently decodable pieces without losing the information needed to reconstruct rows, nulls, and nested structure?**

Parquet's answer is the page.

## Why pages exist at all

A column chunk could, in theory, be stored as one huge blob. But that would be a poor design:

- decompression would require large monolithic reads,
- selective skipping would be coarse,
- statistics would only exist at chunk granularity,
- memory usage during decode would spike,
- incremental writing would be awkward.

Pages solve those problems by creating smaller physical units inside a column chunk.

Each page can carry:

- its own header,
- its own value count,
- its own encoding details,
- its own optional statistics,
- and its own physical byte range.

So pages are the format's answer to “how small can we make a meaningful decode unit?”

## The three page kinds that matter first

`arrow-rs` models:

- `DictionaryPage`
- `DataPage` (V1)
- `DataPageV2`

The dictionary page supplies an indirection table when dictionary encoding is used. The data pages carry the actual leaf stream for the chunk.

## Why page headers are per-page instead of per-chunk

Because decoding decisions are page-local.

A reader may need to know, for *this page*:

- how many values it contains,
- what value encoding was used,
- whether statistics are present,
- for V2, how long the rep/def level regions are,
- whether the payload bytes are compressed.

If that information only existed once at the chunk level, page-local heterogeneity and skipping would be much harder.

So the format pays some metadata overhead to keep pages self-describing enough to decode independently.

## Dictionary page: why it exists

Dictionary encoding turns repeated values into small integer codes.

That only works if the reader first sees the dictionary itself. Hence the format convention:

- optional dictionary page first,
- then data pages whose values may be dictionary-encoded.

This is why a chunk's byte range often begins at the dictionary page offset if one exists.

## Data Page V1 layout

Conceptually:

```text
PageHeader {
  type = DATA_PAGE
  compressed_page_size
  uncompressed_page_size
  data_page_header {
    num_values
    encoding
    definition_level_encoding
    repetition_level_encoding
  }
}
page body:
  [rep levels][def levels][encoded values]
```

The crucial fact is that V1 stores the decoding recipe for levels inside the V1 page header.

## Data Page V2 layout

Conceptually:

```text
PageHeader {
  type = DATA_PAGE_V2
  compressed_page_size
  uncompressed_page_size
  data_page_header_v2 {
    num_values
    num_nulls
    num_rows
    repetition_levels_byte_length
    definition_levels_byte_length
    encoding
    is_compressed
  }
}
page body:
  [rep levels][def levels][encoded values]
```

The V2 design is answering a more explicit question:

**Can we make the boundary between level bytes and value bytes explicit so the reader does not have to infer it from a secondary encoding contract?**

That is why V2 records:

- the level section lengths,
- row count,
- null count,
- and compression semantics more explicitly.

## Why V2 needed explicit level lengths

In a nested format, the reader must split the page body into:

- repetition level bytes,
- definition level bytes,
- encoded value bytes.

V2 makes those lengths first-class header fields. This reduces ambiguity and allows the reader to treat the leading level region specially.

One important consequence: the level streams may remain uncompressed even when values are compressed. That is why `DataPageV2` tracks `is_compressed` and explicit byte lengths.

## `CompressedPage` versus `Page`

`Page` is the logical decoded page shape.
`CompressedPage` is the write-time form whose buffer may still be compressed.

That split is essential because the writer must reason about two different truths at once:

- logical page meaning: number of values, encodings, stats,
- physical emitted bytes: compressed size, uncompressed size, payload bytes.

If you blur those together, metadata accounting becomes error-prone.

## Level streams are real byte streams, not hidden state

This is one of the easiest points to underestimate.

Definition and repetition levels are not “implied by the schema during read.” They are explicitly encoded into the page body.

`LevelEncoder` in `arrow-rs` makes this concrete:

- V1 reserves a 4-byte length prefix in front of the RLE data,
- V2 writes the raw RLE bytes expected by the V2 contract.

The schema tells you the *maximum* level values. The page body carries the *actual* level sequence for the encoded records.

## Why level encoding is RLE-friendly

Nested data often contains long stretches of repeated structural states:

- many required values,
- many non-null values,
- many repeated elements in a list,
- many empty or null runs.

That makes repetition and definition levels good candidates for compact run-length-style encoding. The `num_required_bits(max_level)` calculation in `LevelEncoder` shows another important property: level streams are bounded by the schema, so their bit width is known ahead of time.

## One page, three logical streams

The right way to think about a data page is:

```text
data page
  = structure stream(s) + value stream

structure stream(s):
  repetition levels
  definition levels

value stream:
  encoded non-null leaf values
```

That decomposition explains several subtle behaviors:

- number of physical entries can differ from number of decoded values,
- number of levels can differ from number of non-null values,
- nested decode must track record completion separately from value count.

## Why page metrics later become footer metadata

As each page is written, the writer learns:

- page offset,
- page type,
- compressed/uncompressed size,
- number of values,
- page-level stats.

Those facts later feed:

- chunk metadata,
- optional page indexes,
- reader skipping logic.

So pages are not only decode units; they are also the units from which later metadata is synthesized.

## Reimplementation checklist

If you were implementing pages from scratch, you would need:

1. a page header format with page-type-specific subheaders,
2. a dictionary page path,
3. V1 and V2 data page handling,
4. level stream encoding/decoding,
5. clear accounting of compressed vs uncompressed sizes,
6. page-local statistics and offsets for later indexing.

## Source markers

- Page enum and logical page kinds:
  [`parquet/src/column/page.rs`](../../parquet/src/column/page.rs#L17)
- `CompressedPage` and thrift header generation:
  [`parquet/src/column/page.rs`](../../parquet/src/column/page.rs#L134)
- Page write metrics:
  [`parquet/src/column/page.rs`](../../parquet/src/column/page.rs#L278)
- Level encoder:
  [`parquet/src/encodings/levels.rs`](../../parquet/src/encodings/levels.rs#L17)
- Serialized page decode path:
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs#L387)
- Column writer page assembly:
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs)
