# 8. IPC: Streams, Files, Framing, And Message Layout

This chapter answers the transport question:

**How do you serialize Arrow memory for files and streams without giving up the ability to reconstruct the same logical buffers cheaply on the other side?**

Arrow IPC is the answer.

## Why Arrow IPC is so tightly coupled to Arrow memory

Arrow IPC is not a second independent representation of tabular data. It is a transport wrapper around the Arrow in-memory model.

The core idea is:

- schema and message metadata are encoded as FlatBuffers,
- array bodies are still serialized in Arrow-friendly buffer order,
- framing and alignment make those bodies reconstructible, and sometimes zero-copy reusable.

So IPC should be thought of as:

```text
Arrow memory
  + message framing
  + metadata encoding
  + alignment / compression rules
```

not as:

```text
a different data model for files
```

## Why there are stream and file variants

These variants solve different discovery problems.

### Streaming format

Optimized for:

- forward-only consumption,
- sockets/pipes,
- indefinite or incremental production.

### File format

Optimized for:

- reopening later,
- random access,
- footer-based discovery.

That is why the same logical Arrow messages have two related but different framing environments.

## Why the format uses magic bytes and continuation markers

Two constants matter immediately:

- `ARROW_MAGIC = "ARROW1"`
- `CONTINUATION_MARKER = [0xff; 4]`

These are not just file signatures. They support:

- framing validation,
- distinguishing metadata/message boundaries,
- reopening file-format IPC from the tail,
- compatibility across readers.

As with Parquet, the file format includes a tail-oriented discovery mechanism because a file reader wants to reopen bytes and find the footer reliably.

## Why alignment appears again at the IPC layer

`IpcWriteOptions` allows alignment `8`, `16`, `32`, or `64`, defaulting to `64`.

This tells you IPC is not trying to be maximally compact at any cost. It is trying to preserve the conditions under which reconstructed buffers can still be interpreted efficiently.

That is the transport-layer version of the same policy we saw in `arrow-buffer`:

- pay some padding,
- preserve predictable aligned bodies,
- make reconstructed arrays cheaper to use.

## High-level file picture

```text
Arrow IPC file

┌────────────┬──────────────┬──────────────┬──────────────┬────────────┐
│ ARROW1     │ schema msg   │ dict/batch   │ footer       │ ARROW1     │
│ magic      │ + body       │ msgs + bodies│ + footer len │ magic      │
└────────────┴──────────────┴──────────────┴──────────────┴────────────┘
```

The exact bytes are more detailed, but this is the right structural model.

## What the writer is really doing

The IPC writer does not “serialize arrays from scratch.” It walks already-valid Arrow arrays and emits:

- schema metadata,
- dictionary messages if needed,
- record batch metadata,
- aligned body buffers in the order expected by the IPC format.

```text
Schema / RecordBatch
        │
        ▼
IpcDataGenerator
  - traverse arrays
  - encode dictionaries
  - build FlatBuffer metadata
  - collect body buffers
        │
        ▼
FileWriter / StreamWriter
  - framing
  - metadata bytes
  - aligned body bytes
```

The critical fact is that the body layout is derived from Arrow array layout, not invented separately by IPC.

## Why the reader needs more than “read bytes and cast”

The reader must solve several problems in order:

- decode framing,
- decode FlatBuffer metadata,
- know how many buffers belong to each field,
- handle variadic cases such as view types,
- reconstruct children recursively,
- apply decompression when needed,
- sometimes enforce alignment.

This is why `RecordBatchDecoder::create_array` is so important. It is the place where transport metadata becomes reconstructed array structure.

## Why view types make the IPC reader more interesting

The reader tracks `variadic_counts` because view types such as `Utf8View` and `BinaryView` can require a variable number of buffers.

This is a good example of Arrow IPC staying faithful to the in-memory model:

- if the memory model allows variadic backing buffers,
- the IPC metadata has to preserve enough information to reconstruct that shape.

## Why null buffers are always represented in IPC

The reader comments explicitly note:

- in IPC, null buffers are always set, but may be empty,
- readers may discard them when null count is zero.

This reveals a useful transport policy:

- use a regular framing/layout contract even when a buffer is logically empty,
- let readers collapse the degenerate case later.

That kind of regularity simplifies serialization rules even if it sometimes carries a little structural redundancy.

## Why `FileDecoder` is the key low-level bridge

`FileDecoder` is where:

- file framing,
- footer discovery,
- schema metadata,
- dictionaries,
- and body buffers

finally become `RecordBatch`es and arrays again.

It is one of the best places to study if you want to build another IPC implementation from scratch.

## Why IPC can be zero-copy friendly

If the file is memory-mapped and the serialized bodies satisfy the required alignment and layout, the decoder can often let arrays point directly into that underlying memory.

This is why the zero-copy mmap example matters so much:

- it demonstrates that IPC is close enough to in-memory Arrow that “deserialization” can be mostly metadata reconstruction plus buffer slicing.

That is the deepest practical reason Arrow IPC is shaped the way it is.

## Reimplementation checklist

To build another Arrow IPC implementation, you need:

1. message framing for stream and file modes,
2. schema metadata encoding/decoding,
3. body buffer serialization in Arrow layout order,
4. alignment and padding rules,
5. dictionary message handling,
6. recursive array reconstruction on read.

That is the transport layer.

## Source markers

- IPC overview and framing constants:
  [`arrow-ipc/src/lib.rs`](../../arrow-ipc/src/lib.rs#L1),
  [`arrow-ipc/src/lib.rs`](../../arrow-ipc/src/lib.rs#L72)
- `IpcWriteOptions`:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L51)
- `schema_to_bytes_with_dictionary_tracker`:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L200)
- File writer magic handling:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L1128),
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L1236)
- Reader reconstruction and variadic buffers:
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L78)
- Footer length and `FileDecoder`:
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L898),
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L977)
- Alignment enforcement:
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L1017)
- Zero-copy example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs#L1)
