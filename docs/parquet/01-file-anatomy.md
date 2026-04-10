# 01. File Anatomy

This chapter answers the most important first-principles question in Parquet:

**What minimum information must a reader recover from raw bytes in order to find the schema, the row groups, the page locations, and the decoding rules?**

Parquet's file layout is the answer to that question.

## Start with the design problem, not the structs

Imagine you are designing a columnar file format for very large datasets.

You want the reader to be able to:

- open the file without scanning the whole thing,
- find row groups quickly,
- skip columns it does not need,
- use min/max/index metadata before reading data pages,
- handle files that may contain optional side structures such as page indexes,
- write data in one forward pass as much as possible.

Those requirements immediately rule out a naive format such as:

```text
[header with every offset][all data]
```

Why? Because many offsets are not known until data pages, column chunks, row groups, and auxiliary structures have already been written.

Parquet therefore chooses the opposite strategy:

- write data first,
- write the master metadata last,
- teach the reader to discover the file from the end.

## The shortest correct mental model

```text
PAR1
  row group 0
    column chunk 0
      dictionary page?      optional
      data pages...
    column chunk 1
      ...
  row group 1
    ...
  optional page indexes / other side structures
  thrift FileMetaData
  4-byte metadata length
PAR1
```

That final 8-byte tail is the bootstrap point for the whole file.

## Why the footer is at the end

This is the central design decision.

The writer cannot know the final values of:

- row group offsets,
- column chunk compressed sizes,
- page index offsets,
- bloom-filter offsets,
- total row counts,
- many statistics and encodings,

until the data has already been emitted.

So Parquet treats the footer as a post-hoc directory of the file.

```text
writer knowledge over time

start of file:
  knows schema
  does NOT know chunk offsets, sizes, page offsets, final footer length

after writing row groups:
  now knows chunk spans and many stats

near end of file:
  can finally serialize the directory of everything written
```

This is why the format feels “inside out” compared with simpler binary formats.

## Full-file byte map

```text
byte 0                                                                 file end
│                                                                           │
├─ PAR1 ─┬─ row group 0 ─┬─ row group 1 ─┬─ ... ─┬─ side structures ─┬─ tail │
│        │               │               │       │                    │       │
│        │               │               │       ├─ column indexes    │       ├─ 4-byte metadata length
│        │               │               │       ├─ offset indexes    │       └─ PAR1 / PARE
│        │               │               │       ├─ bloom filters     │
│        │               │               │       └─ thrift FileMetaData
│        │               │               │
│        │               │               └─ one column chunk per leaf column
│        │               │
│        │               └─ one column chunk per leaf column
│        │
│        └─ one column chunk per leaf column
```

Two things matter here:

1. the file is physically organized by row group and then by leaf column,
2. the footer is a directory that points back into earlier bytes.

## The final 8 bytes are the reader's entry point

`arrow-rs` models the final 8 bytes with `FooterTail`:

```text
+----------------------+------------------+
| 4-byte little-endian | 4-byte magic     |
| metadata length      | 'PAR1' or 'PARE' |
+----------------------+------------------+
```

A reader that cannot parse these 8 bytes cannot even know where the metadata begins.

The algorithm is:

1. read last 8 bytes,
2. verify magic,
3. decode metadata length,
4. jump backward by that many bytes,
5. decode the thrift `FileMetaData`.

This is why even remote readers can often start with a tiny tail fetch.

## Why the magic appears at both start and end

The starting `PAR1` and trailing `PAR1` are not redundant decoration.

They help with:

- quick file identification,
- corruption detection,
- sanity-checking when seeking near either boundary,
- distinguishing a valid Parquet file from a random byte blob with a plausible footer length.

The ending magic is especially important because the reader enters from the tail. The leading magic is still useful for validation and conventional format identification.

## Why metadata is partly contiguous and partly offset-based

Parquet metadata is not just one big self-contained footer object.

Some structures, such as page indexes, may be written outside the contiguous `FileMetaData` blob. The footer then stores offsets to those structures.

Why choose that shape?

- page indexes may be produced late,
- they may be large,
- they are optional,
- keeping them separate avoids forcing one giant monolithic metadata blob design.

So the footer is better understood as a root directory than a complete copy of every metadata object.

## What is physically inside a row group

A row group is the file's main horizontal partition. It answers:

**Which subset of rows can be processed together while still storing each leaf column contiguously?**

Inside one row group:

- each leaf column gets one column chunk,
- each column chunk is a sequence of pages,
- pages are the smallest independently decodable physical units.

```text
row group
├─ column chunk: leaf column 0
│  ├─ dictionary page?      optional
│  ├─ data page 0
│  ├─ data page 1
│  └─ data page N
├─ column chunk: leaf column 1
│  └─ pages...
└─ column chunk: leaf column 2
   └─ pages...
```

This is already enough to explain why Parquet is good at column pruning: unwanted leaf columns can be skipped at the column chunk level.

## Why `byte_range()` is so important

At some point the metadata must become actual IO.

`ColumnChunkMetaData::byte_range()` answers the practical question:

**What exact byte interval do I need to fetch to read this column chunk?**

It resolves that by choosing:

- dictionary page offset if one exists,
- otherwise first data page offset,
- and pairing that with the recorded compressed chunk length.

That is the bridge between logical metadata and physical reads.

## Reimplementation checklist

If you had to write a minimal compatible reader from scratch, this chapter says you would need:

1. a way to read the final 8 bytes,
2. code to validate `PAR1` / `PARE`,
3. little-endian decoding of metadata length,
4. thrift decoding of `FileMetaData`,
5. logic to turn chunk metadata into byte ranges.

That is the true “file anatomy” boundary.

## Source markers

- File-level constants:
  [`parquet/src/file/mod.rs`](../../parquet/src/file/mod.rs#L111)
- Footer tail parsing:
  [`parquet/src/file/metadata/footer_tail.rs`](../../parquet/src/file/metadata/footer_tail.rs#L17)
- Metadata reader and tail-first parsing:
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L31),
  [`parquet/src/file/metadata/reader.rs`](../../parquet/src/file/metadata/reader.rs#L222)
- File writer startup and footer assembly:
  [`parquet/src/file/writer.rs`](../../parquet/src/file/writer.rs#L145),
  [`parquet/src/file/writer.rs`](../../parquet/src/file/writer.rs#L176)
- Metadata writer and final footer layout:
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L52),
  [`parquet/src/file/metadata/writer.rs`](../../parquet/src/file/metadata/writer.rs#L184)
- Column chunk byte span helper:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L1027)
