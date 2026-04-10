# Arrow IPC Deep Dive

This tutorial set is a focused deep dive into Arrow IPC in `arrow-rs`.

It assumes you already understand the Arrow memory model, arrays, buffers, and schemas from the Arrow docs in this repository, especially:

- `docs/arrow/02-memory-buffers-and-alignment.md`
- `docs/arrow/03-arraydata-and-physical-layout.md`
- `docs/arrow/08-ipc-files-streams-and-message-layout.md`
- `docs/arrow/12-worked-example-from-arrays-to-ipc.md`

This set does **not** repeat all of Arrow from scratch. Instead, it answers the narrower but deeper question:

> How does Arrow serialize schemas, dictionaries, record batches, and framing metadata into a portable IPC stream or file, and how does `arrow-ipc` in this repository implement that contract?

Read in order:

1. [01-why-ipc-and-what-it-standardizes.md](./01-why-ipc-and-what-it-standardizes.md)
2. [02-message-model-and-flatbuffer-metadata.md](./02-message-model-and-flatbuffer-metadata.md)
3. [03-stream-vs-file-framing.md](./03-stream-vs-file-framing.md)
4. [04-schema-dictionaries-and-versioning.md](./04-schema-dictionaries-and-versioning.md)
5. [05-record-batch-body-layout.md](./05-record-batch-body-layout.md)
6. [06-reading-and-decoding.md](./06-reading-and-decoding.md)
7. [07-writing-and-encoding.md](./07-writing-and-encoding.md)
8. [08-compression-alignment-and-zero-copy.md](./08-compression-alignment-and-zero-copy.md)
9. [09-usage-patterns-and-worked-examples.md](./09-usage-patterns-and-worked-examples.md)

## Core Mental Model

Arrow IPC is not "serialize a `RecordBatch` into one blob."

It is:

```text
framing
  +
flatbuffer metadata message
  +
body buffers laid out in protocol order
  +
stream/file-level state
```

The key split is:

- metadata tells you *how to interpret* the body,
- body carries the actual buffers,
- stream/file framing tells you *where each message begins and ends*.

## Main Source Markers

- `arrow-ipc/src/lib.rs`
- `arrow-ipc/src/convert.rs`
- `arrow-ipc/src/writer.rs`
- `arrow-ipc/src/reader.rs`
- `arrow-ipc/src/reader/stream.rs`
- `arrow-ipc/src/compression.rs`
- `arrow-ipc/tests/test_delta_dictionary.rs`

## What To Pay Attention To

When reading these docs, keep these implementation questions in mind:

1. Why are there separate stream and file formats?
2. Why does the schema live in flatbuffer metadata instead of the body?
3. Why are dictionaries stateful across messages?
4. Why does the reader rebuild arrays from `FieldNode`s and `Buffer`s instead of deserializing row by row?
5. Why do alignment and padding exist everywhere in the writer?
6. Why can the file format offer random access but the stream format cannot?
