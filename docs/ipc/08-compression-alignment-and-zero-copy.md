# 8. Compression, Alignment, and Zero-Copy

This chapter covers the low-level policies that make IPC practical rather than merely correct.

## Alignment

Alignment is exposed directly in `IpcWriteOptions` and used throughout `writer.rs`.

Allowed values are:

- 8
- 16
- 32
- 64

Default is 64.

### Why Alignment Is Configurable

Because Arrow buffers benefit from predictable typed-memory alignment, but different environments may care differently about:

- strictness,
- legacy compatibility,
- performance tradeoffs.

The default of 64 is conservative and performance-oriented.

## Why Padding Is Everywhere

Padding appears in several places:

- after metadata messages,
- after body buffers,
- after magic header in files.

This can look wasteful until you remember the real goal: preserve aligned access and stable offsets across languages and runtimes.

IPC is not optimizing for minimum byte count at all costs. It is optimizing for faithful and efficient Arrow reconstruction.

## Reader Alignment Choices

On read, the decoder can:

- require already-aligned buffers,
- or realign by copying.

This policy is visible in:

- `RecordBatchDecoder::with_require_alignment`
- `StreamDecoder::with_require_alignment`
- `FileDecoder::with_require_alignment`

That design is good because it makes the zero-copy boundary explicit instead of pretending all inputs are equally safe for direct typed access.

## Compression Format

`arrow-ipc/src/compression.rs` documents the exact per-buffer layout:

```text
[8 bytes]: uncompressed length
[rest]:    compressed data
```

Special case:

- uncompressed length `-1` means the data is stored uncompressed even though the batch is in a compression-enabled mode.

### Why This Fallback Exists

Because compression is not always a win.

If compressed output is larger than input, the writer keeps the raw data and marks it accordingly. That is a pragmatic policy that preserves correctness while avoiding counterproductive compression.

## Buffer-Oriented Compression

The key policy is that compression is applied to body buffers, not to the metadata flatbuffer and not to the whole file/stream as one opaque blob.

That makes sense because:

- the metadata must stay directly readable,
- buffer boundaries remain meaningful,
- decompression can stay coupled to the buffer iterator in the reader.

See `read_buffer` in `reader.rs` for the reverse path.

## Zero-Copy Reality

IPC is often described as enabling zero-copy interchange. That is true, but only conditionally.

It is most accurate to say:

- IPC is *designed to make zero-copy possible when alignment and representation allow it*.

Things that can force copies:

- misaligned input,
- decompression,
- validation/normalization paths,
- some type-specific reconstructions.

That is why the reader API talks explicitly about alignment rather than promising unconditional zero-copy.

## Endianness and Zero-Copy

The endianness check is part of the same story.

If source and target endianness differ, straightforward zero-copy typed interpretation no longer works safely. Rejecting mismatched endianness is therefore consistent with IPC's broader design philosophy.

## Source Markers

- `arrow-ipc/src/writer.rs`: alignment handling, padding, `write_message`
- `arrow-ipc/src/reader.rs`: `read_buffer`, alignment policy on decode
- `arrow-ipc/src/compression.rs`: buffer compression/decompression format and context reuse
- `arrow-ipc/src/lib.rs`: endianness helper

## Practical Summary

Alignment and compression are not side features. They are the main reason IPC can be both:

- faithful to Arrow memory layout,
- and practical for portable transport/storage.
