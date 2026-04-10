# 6. Reading and Decoding

The reader side of IPC is where all prior design decisions become visible:

- framing decisions,
- message typing,
- dictionary state,
- compression metadata,
- alignment policy,
- projection.

`arrow-ipc` separates the problem into several layers, and that separation is one of the crate's strongest design choices.

## File Reading

`FileReaderBuilder::build` in `arrow-ipc/src/reader.rs` does the outer file work:

1. seek to the last 10 bytes,
2. read `[footer_len][ARROW_MAGIC]`,
3. validate magic,
4. seek backwards to the footer,
5. parse footer flatbuffer,
6. read schema,
7. read dictionary blocks,
8. prepare record-batch block index.

That is the complete random-access bootstrap sequence.

### Why Dictionaries Are Read Before Batches

Because later record batches may depend on them by id.

The file footer explicitly separates dictionary blocks from record-batch blocks, which makes reader bootstrap deterministic.

## `FileDecoder`

`FileDecoder` handles low-level batch and dictionary decoding once the outer file has supplied a `Block` and raw bytes.

It keeps:

- schema,
- dictionary cache,
- metadata version,
- optional projection,
- alignment/validation policy.

This is the right abstraction boundary: the outer reader handles file navigation, the decoder handles semantic decoding.

## Streaming Read

`StreamReader::try_new` and `MessageReader` in `arrow-ipc/src/reader.rs`, plus the lower-level `StreamDecoder` in `arrow-ipc/src/reader/stream.rs`, handle stream format.

There are two related but different models here:

- `StreamReader`: pull-based iterator over `RecordBatch`
- `StreamDecoder`: push-based incremental decoder over arbitrary input buffers

### Why Both APIs Exist

The iterator API is convenient when you already have a `Read`.

The push-based decoder is necessary when data arrives as arbitrary chunks from sources like:

- object storage range streams,
- chunked network input,
- evented runtimes handing off byte slabs.

This is a very practical design choice and worth copying in any rewrite.

## `StreamDecoder` State Machine

`arrow-ipc/src/reader/stream.rs` defines:

- `Header`
- `Message`
- `Body`
- `Finished`

Diagram:

```text
Header
  -> parse continuation marker / size
  -> Message
Message
  -> collect metadata flatbuffer bytes
  -> Body
Body
  -> collect body bytes
  -> dispatch by header_type
  -> Header
Finished
```

This state machine exists because network or chunked inputs do not promise message boundaries aligned with IPC boundaries.

## `RecordBatchDecoder`

This is the core semantic decoder in `arrow-ipc/src/reader.rs`.

It binds together:

- record batch flatbuffer metadata,
- body buffer,
- schema,
- dictionary cache,
- compression info,
- projection,
- alignment policy.

The decoder then performs a recursive read mirroring the writer's recursive traversal.

## Alignment Policy

The reader exposes `with_require_alignment`.

If alignment is required:

- misaligned buffers are rejected.

If not:

- the decoder may allocate aligned buffers and copy as needed.

This is the right policy split because some consumers want strict zero-copy guarantees while others prefer permissive decoding.

## Validation Policy

`FileDecoder` and `StreamReader` expose unsafe skip-validation modes.

Why this exists:

- validation costs CPU,
- trusted self-produced data may justify skipping it,
- but skipping validation on untrusted data is dangerous.

The code comments in `reader.rs` are appropriately cautious here.

## Unsupported or Unexpected Messages

The readers are careful about protocol state:

- stream must begin with a schema,
- unexpected schema mid-stream is an error,
- unsupported header types are rejected,
- buffer/node mismatches become parse/schema errors.

This defensive posture is necessary because IPC parsing is close to raw memory layout reconstruction.

## Source Markers

- `arrow-ipc/src/reader.rs`: `FileReaderBuilder`, `FileReader`, `FileDecoder`, `StreamReader`, `RecordBatchDecoder`
- `arrow-ipc/src/reader/stream.rs`: `StreamDecoder` state machine
- `arrow-ipc/src/compression.rs`: buffer decompression details

## Reimplementation Lesson

A robust IPC reader should not be one big `read_batch` function.

It should separate:

1. outer framing/file navigation,
2. message parsing,
3. semantic state like schema/dictionaries,
4. array reconstruction.

That is exactly what `arrow-ipc` does.
