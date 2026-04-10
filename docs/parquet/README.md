# Parquet Deep Dive In `arrow-rs`

This directory is a guided, implementation-linked tutorial for understanding Parquet in the `arrow-rs` repository at a level where you should be able to:

1. explain the file format from bytes upward,
2. justify why the format is organized this way,
3. answer low-level questions about nested encoding, page boundaries, and footer discovery,
4. follow the `arrow-rs` reader and writer with confidence,
5. sketch or reimplement a compatible reader/writer from scratch.

The earlier version of these notes leaned too much toward source summarization. This version is meant to be read like a format lecture for systems engineers: every structure is described as an answer to a problem the format must solve.

## The questions this tutorial is trying to answer

If you cannot answer these questions, you do not yet understand Parquet deeply enough:

- Why is the footer at the end of the file instead of the beginning?
- Why does Parquet store data by leaf column instead of by logical row?
- Why are row groups needed as a separate unit from pages?
- Why are repetition and definition levels necessary for nested data?
- Why are there separate page headers instead of one giant chunk header?
- Why does the writer discover many offsets only after writing data?
- Why can page indexes live outside the contiguous footer blob?
- Why can a reader reconstruct rows without ever seeing a row-oriented byte layout?

The chapter order is designed to answer those in dependency order.

## Recommended reading order

1. [01-file-anatomy.md](./01-file-anatomy.md)
2. [02-schema-and-nesting.md](./02-schema-and-nesting.md)
3. [03-metadata-and-indexes.md](./03-metadata-and-indexes.md)
4. [04-pages-and-encodings.md](./04-pages-and-encodings.md)
5. [05-read-path.md](./05-read-path.md)
6. [06-write-path.md](./06-write-path.md)
7. [07-apis-and-tools.md](./07-apis-and-tools.md)
8. [08-worked-example-from-rows-to-pages.md](./08-worked-example-from-rows-to-pages.md)

## How to read these notes

Each chapter has a `Source markers` section. Those markers point at the implementation entry points in this repository. Use them in this order:

1. read the conceptual explanation first,
2. check the diagram until you can restate it in your own words,
3. only then open the linked code and map the concepts onto actual structs and functions.

If you open the code too early, Parquet can look like “a lot of metadata structs and readers.” In reality it is a very tight answer to a specific set of storage and reconstruction constraints.

## Mental model of the whole format

At the highest level, a Parquet file is a contract between a writer that *shreds* logical rows into columnar streams and a reader that must later *reassemble* those logical rows.

```text
logical rows
   │
   ├─ shred nested structure into leaf columns
   ├─ record null / repetition information as levels
   ├─ split each leaf column into pages
   ├─ group pages into row groups
   └─ write enough metadata so a reader can find and decode all of it later
        │
        ▼
      file bytes
        │
        ▼
reader starts from footer
   ├─ discovers schema, row groups, offsets, stats, indexes
   ├─ chooses needed row groups / columns / pages
   ├─ decodes level streams + values
   └─ reconstructs logical rows or Arrow arrays
```

That is the whole format in one picture.

## Core source map

- File-level constants and crate overview:
  [`parquet/src/lib.rs`](../../parquet/src/lib.rs),
  [`parquet/src/file/mod.rs`](../../parquet/src/file/mod.rs)
- Schema model and flattening:
  [`parquet/src/schema/types.rs`](../../parquet/src/schema/types.rs)
- Metadata model and footer handling:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs),
  [`parquet/src/file/metadata/footer_tail.rs`](../../parquet/src/file/metadata/footer_tail.rs),
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs),
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs)
- Pages, level streams, and chunk IO:
  [`parquet/src/column/page.rs`](../../parquet/src/column/page.rs),
  [`parquet/src/column/reader.rs`](../../parquet/src/column/reader.rs),
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs),
  [`parquet/src/encodings/levels.rs`](../../parquet/src/encodings/levels.rs)
- File reader and writer:
  [`parquet/src/file/reader.rs`](../../parquet/src/file/reader.rs),
  [`parquet/src/file/serialized_reader.rs`](../../parquet/src/file/serialized_reader.rs),
  [`parquet/src/file/writer.rs`](../../parquet/src/file/writer.rs)
- Arrow integration:
  [`parquet/src/arrow/mod.rs`](../../parquet/src/arrow/mod.rs),
  [`parquet/src/arrow/arrow_reader/mod.rs`](../../parquet/src/arrow/arrow_reader/mod.rs),
  [`parquet/src/arrow/arrow_writer/mod.rs`](../../parquet/src/arrow/arrow_writer/mod.rs)
- Examples and CLI tools:
  [`parquet/examples`](../../parquet/examples),
  [`parquet/src/bin`](../../parquet/src/bin)
