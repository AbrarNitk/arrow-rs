# 8. IPC: Streams, Files, Framing, And Message Layout

Arrow IPC is how Arrow memory is serialized without abandoning the Arrow memory model.

The key idea is:

- the logical schema and record batches are encoded as FlatBuffers metadata,
- the actual Arrow buffers remain laid out in Arrow-friendly memory order,
- file and stream variants add different framing rules.

## Two IPC variants

`arrow-ipc` supports:

- streaming format: append-only, sequential consumption,
- file format: random-access friendly, footer-based discovery.

The top-level crate docs say this directly, and the reader/writer modules implement both forms.

## Magic bytes and continuation markers

Two low-level constants define a lot of the format mechanics:

- `ARROW_MAGIC = "ARROW1"`
- `CONTINUATION_MARKER = [0xff; 4]`

These are the recognizable sentinels the reader and writer use to frame and validate messages.

## High-level file picture

```text
Arrow IPC file

┌────────────┬──────────────┬──────────────┬──────────────┬────────────┐
│ ARROW1     │ schema msg   │ dict/batch   │ footer       │ ARROW1     │
│ magic      │ + body       │ msgs + bodies│ + footer len │ magic      │
└────────────┴──────────────┴──────────────┴──────────────┴────────────┘
```

The exact internal structure uses FlatBuffer metadata and aligned Arrow bodies, but this sketch is enough to orient yourself.

## Why alignment matters again in IPC

`IpcWriteOptions` allows alignment values `8`, `16`, `32`, or `64`, and defaults to `64`.

That tells you the writers are not merely dumping arbitrary byte streams. They are trying to preserve alignment properties that help readers reinterpret the serialized buffers efficiently and, in some cases, zero-copy.

## What the writer really emits

The writer path roughly does this:

```text
Schema / RecordBatch
        │
        ▼
IpcDataGenerator
  - walk arrays
  - encode dictionaries
  - build FlatBuffer metadata
  - collect Arrow body buffers
        │
        ▼
FileWriter / StreamWriter
  - write framing
  - write metadata
  - write aligned buffer bodies
```

The important detail is that the body layout is derived from Arrow array layout, not independently invented by IPC.

## Reader path

The reader reverses that process:

```text
bytes
  │
  ▼
frame parsing
  │
  ▼
FlatBuffer message decode
  │
  ▼
schema / dictionary reconstruction
  │
  ▼
ArrayData / typed arrays / RecordBatch
```

`FileDecoder` is the low-level bridge here. It takes decoded metadata plus raw byte regions and turns them back into Arrow arrays.

## Why IPC can be zero-copy friendly

If a file is memory-mapped and the buffers are already aligned and valid, the decoder can often hand out Arrow arrays that reference the underlying bytes directly. The example in `arrow/examples/zero_copy_ipc.rs` shows exactly that workflow.

This is why the Arrow in-memory format and IPC format are so closely related. IPC is designed to preserve reusability of the same memory layout.

## Compression, dictionaries, and versioning

`IpcWriteOptions` also shows three policy choices:

- metadata version is explicit,
- compression is allowed only when compatible with the metadata version,
- dictionary handling is configurable.

That makes IPC a controlled transport boundary rather than a blind dump of structs.

## File versus stream intuition

```text
Stream
  schema -> dictionaries? -> batch -> batch -> batch -> end
  optimized for forward consumption

File
  magic -> messages -> footer -> footer length -> magic
  optimized for reopening and random access
```

Use the stream format when transport order matters and seeking is unavailable. Use the file format when the bytes will be stored and reopened later.

## Source markers

- IPC crate overview:
  [`arrow-ipc/src/lib.rs`](../../arrow-ipc/src/lib.rs#L1)
- Format sentinels:
  [`arrow-ipc/src/lib.rs`](../../arrow-ipc/src/lib.rs#L72)
- Write options and default alignment:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L51),
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L152)
- File writer magic handling:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L1128),
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L1236)
- Continuation marker writes:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L1642)
- Reader and `FileDecoder`:
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L977)
- Zero-copy example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs#L1)
