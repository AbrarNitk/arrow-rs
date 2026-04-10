# 7. Writing and Encoding

This chapter is the writer-side counterpart to the previous one.

The central writer insight is:

> IPC writing is mostly about maintaining the right state and invariants while translating Arrow arrays into message metadata plus aligned body buffers.

## `IpcWriteOptions`

`IpcWriteOptions` in `arrow-ipc/src/writer.rs` captures the major writer policies:

- alignment,
- legacy vs modern framing,
- metadata version,
- optional compression,
- dictionary handling.

This is exactly what should be configurable. These are transport/format policy choices, not logical schema choices.

## `IpcDataGenerator`

`IpcDataGenerator` handles low-level encoding of:

- schema messages,
- dictionary batches,
- record batches.

It is the core reusable encoder used by both file and stream writers.

This is a strong design because it avoids duplicating array serialization logic across `FileWriter` and `StreamWriter`.

## Schema First

Both file and stream writers emit schema immediately at construction time.

Why:

- every subsequent message depends on it,
- readers expect it before batches,
- schema is the stable contract of the stream/file.

This is one of the strongest invariants in the whole format.

## `StreamWriter`

`StreamWriter` is the simpler outer container:

- write schema on initialization,
- on each batch, write needed dictionaries then batch,
- on finish, write EOS continuation.

It does not need to remember previous batch offsets or build a footer.

## `FileWriter`

`FileWriter` adds several file-only responsibilities:

- write magic header,
- maintain `block_offsets`,
- accumulate `dictionary_blocks`,
- accumulate `record_blocks`,
- serialize a footer at the end,
- write footer length and trailing magic.

This is exactly why file writing is more than "stream writing plus a different suffix." Random access requires index bookkeeping from the first write onward.

## Why `write_message` Is a Separate Function

`write_message` in `arrow-ipc/src/writer.rs` is the critical framing boundary.

It handles:

- continuation/length prefix,
- alignment of flatbuffer metadata region,
- metadata bytes,
- body bytes,
- padding.

Keeping this separate is important because:

- both stream and file writers need it,
- framing logic is distinct from array serialization logic,
- misplacing alignment or prefix rules here would corrupt all messages uniformly.

## Dictionary State on Write

The writer's `DictionaryTracker` decides, for each dictionary-encoded field:

- emit nothing,
- emit a new dictionary,
- emit a replacement dictionary,
- emit a delta dictionary.

This means the writer is not stateless with respect to prior batches. That is especially obvious in streams, where stable dictionary ids and incremental updates reduce bandwidth.

## Compression State

Compression is configured per batch through `IpcWriteOptions` and executed via `CompressionContext`.

Why there is a reusable context object:

- codecs like zstd can benefit from reusable state,
- reinitializing compressor state per buffer would waste work,
- the writer naturally processes many buffers over time.

This is a good example of performance engineering living in the right layer: not in array logic, not in outer framing, but in the body-buffer transform layer.

## Writer Invariants

The writer enforces several important invariants:

- alignment must be valid,
- compressed IPC requires V5 metadata,
- file writer cannot write after finish,
- stream writer cannot write after finish,
- arrow body data must be aligned before `write_message`.

These checks matter because IPC corruption is easy to create if you let these invariants drift.

## Source Markers

- `arrow-ipc/src/writer.rs`: `IpcWriteOptions`, `IpcDataGenerator`, `FileWriter`, `StreamWriter`, `write_message`
- `arrow-ipc/src/compression.rs`: reusable compression context and buffer compression policy

## Practical Summary

The writer side is easiest to understand in layers:

```text
Arrow arrays / RecordBatch
  ->
IpcDataGenerator
  ->
EncodedData { ipc_message, arrow_data }
  ->
write_message framing
  ->
stream/file outer container
```

That is the architecture to preserve if you ever rewrite it.
