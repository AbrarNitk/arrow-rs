# 1. Why IPC Exists and What It Standardizes

Arrow IPC exists to solve one precise problem:

> How can two processes exchange Arrow data without collapsing it into rows, text, or some language-specific object graph?

This is a narrower problem than "data storage" and a different problem from "remote RPC." That is why Arrow IPC sits beside Parquet and Flight rather than replacing them.

## The Problem IPC Solves

Arrow arrays already define an in-memory format:

- contiguous buffers,
- explicit null bitmaps,
- offsets for variable-width values,
- child arrays for nested types,
- dictionary indirection where useful.

But in-memory format alone is not enough for exchange. Another process still needs:

- schema identity,
- a stable message boundary,
- a way to find the body buffers,
- dictionary ids and updates,
- versioning rules,
- optional file-level indexing.

IPC is the canonical answer to those needs.

## What IPC Does Not Try To Be

IPC is not optimized for long-term storage the way Parquet is.

It is also not a transport protocol like Flight.

The division of responsibilities is:

```text
Arrow IPC
  serialization contract for Arrow data and schema messages

Parquet
  analytical storage format with encodings, pages, row groups, statistics

Flight
  RPC/data-transfer protocol that often embeds Arrow IPC messages
```

This separation is why Flight can carry Arrow IPC inside `FlightData`, and why Parquet conversion can go through `RecordBatch`es without pretending Parquet and IPC are the same thing.

## The Key Standardization Decision

Arrow IPC standardizes two things at once:

1. the *metadata representation* using FlatBuffers
2. the *body buffer ordering and framing rules*

That is the central design decision.

FlatBuffers makes sense for metadata because it gives:

- cross-language schemas,
- compact structured messages,
- fast access to typed metadata fields.

The body remains raw Arrow buffers because:

- Arrow buffers are already the canonical physical representation,
- copying them into another nested metadata format would defeat the purpose,
- the whole point is to stay close to Arrow's physical layout.

## Why There Are Two IPC Variants

The crate docs in `arrow-ipc/src/lib.rs` state the two variants directly:

- streaming format
- file format

That split exists because a single layout cannot simultaneously optimize for:

- write-once streaming from a socket or pipe
- random access to already-written batches in a seekable file

The stream format is append-friendly.
The file format is seek-friendly.

They reuse the same message model, but package it differently.

## The Minimal IPC Stack

```text
Arrow logical schema
  ->
flatbuffer Schema / Message metadata
  +
Arrow body buffers
  ->
framed stream messages or file blocks
```

Every interesting implementation detail in `arrow-ipc` comes from preserving this split cleanly.

## Why `arrow-ipc` Is Its Own Crate

The crate structure reflects a good architectural boundary:

- `arrow-schema` defines schema types
- `arrow-array` defines arrays and batches
- `arrow-data` handles lower-level array data construction/validation
- `arrow-ipc` handles serialization/deserialization contract

That separation makes sense because IPC is neither purely schema logic nor purely array logic. It is a translation layer between Arrow values and portable bytes.

## Source Markers

- `arrow-ipc/src/lib.rs`: crate-level format split and constants
- `arrow-ipc/src/convert.rs`: schema <-> flatbuffer conversion
- `arrow-ipc/src/writer.rs`: message construction and framing
- `arrow-ipc/src/reader.rs`: message parsing and array reconstruction

## What Comes Next

Now that the role of IPC is clear, the next question is:

> What exactly is a single IPC message, and what is metadata versus body?

That is the subject of the next chapter.
