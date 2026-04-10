# 5. Record Batch Body Layout

This chapter explains how `arrow-ipc` turns a `RecordBatch` into the pair:

- metadata describing nodes and buffers,
- contiguous body bytes carrying those buffers.

## Writer-Side View

`record_batch_to_bytes` in `arrow-ipc/src/writer.rs` builds four main things:

- `nodes`
- `buffers`
- `arrow_data`
- `variadic_buffer_counts`

The writer walks each column, recursively serializes its `ArrayData`, and appends the corresponding logical descriptors.

## Why the Body Is Just Concatenated Buffers

Because Arrow arrays are already represented as buffers.

The IPC writer does not "serialize values" one by one in the general case. It records where each existing Arrow buffer lands in the message body and copies or appends buffer contents in protocol order.

That is much closer to memory imaging than row serialization, though it still requires protocol-aware metadata and sometimes normalization.

## `FieldNode`s: Logical Tree Shape

Each node records:

- logical length,
- null count.

Why this is needed even when buffers exist:

- child traversal for nested types is structural, not implicit in byte offsets alone,
- null count is useful for validation and buffer interpretation,
- some arrays have no body buffers of their own but still occupy a logical node in the traversal.

## `Buffer`s: Physical Slices

Each flatbuffer `Buffer` entry records:

- offset in the body,
- length.

The reader will later iterate over these in the exact order required by the schema traversal.

This is why writer and reader are tightly coupled on traversal order. If either side changes ordering rules, the whole format breaks.

## Recursive Array Writing

`write_array_data` in `arrow-ipc/src/writer.rs` is where the array tree becomes nodes and buffers.

The recursion handles:

- primitive arrays,
- variable-width arrays,
- structs,
- lists and maps,
- unions,
- dictionaries,
- run-end encoded arrays,
- view types.

This recursion is unavoidable because Arrow's type graph is recursive.

## Why View Types Need `variadicBufferCounts`

View types like `Utf8View` and `BinaryView` can have a variable number of buffers beyond the fixed null/view buffers.

That is why the writer records `variadicBufferCounts`, and the reader later consumes them from a queue when decoding. See:

- `append_variadic_buffer_counts` in `writer.rs`
- `create_array` in `reader.rs`

This is a good example of IPC evolving with newer Arrow types while keeping the same overall metadata/body split.

## Body Padding and Alignment

After the body is assembled, the writer pads it to the configured alignment.

Why:

- Arrow buffers want predictable alignment for efficient typed access,
- some consumers may require aligned buffers for zero-copy reconstruction,
- message boundaries need to preserve those guarantees.

`IpcWriteOptions.alignment` and `pad_to_alignment` in `writer.rs` enforce this.

## Compression Placement

Compression in IPC is buffer-oriented, not "compress the whole message blob."

The body compression metadata is attached to the record batch, and each constituent buffer may be stored with an uncompressed-size prefix and compressed payload. See `arrow-ipc/src/compression.rs` and `read_buffer` in `reader.rs`.

Why this granularity makes sense:

- preserves the metadata/body separation,
- keeps buffer boundaries meaningful,
- allows uncompressed fallback when compression is not beneficial.

## Reader-Side Reconstruction

`RecordBatchDecoder` in `arrow-ipc/src/reader.rs` reverses the process by walking:

- schema fields,
- node iterator,
- buffer iterator,
- optional variadic counts,
- dictionary cache.

It reconstructs `ArrayDataBuilder`s for each array type and then builds arrays and the enclosing `RecordBatch`.

Diagram:

```text
schema fields traversal
  +
FieldNode iterator
  +
Buffer iterator
  +
dictionary cache
  +
variadic buffer counts
  ->
ArrayData builders
  ->
typed arrays
  ->
RecordBatch
```

## Why Projection Can Work Without Full Materialization

`RecordBatchDecoder::with_projection` lets the reader skip non-projected fields.

That works because the metadata still describes traversal order precisely. The decoder can:

- consume and skip the corresponding nodes/buffers,
- only build arrays for selected columns,
- project the schema accordingly.

That is a strong example of why the metadata is structured instead of being an opaque schema blob.

## Source Markers

- `arrow-ipc/src/writer.rs`: `record_batch_to_bytes`, `write_array_data`, `append_variadic_buffer_counts`
- `arrow-ipc/src/reader.rs`: `RecordBatchDecoder`, `create_array`, `skip_field`, projection support
- `arrow-ipc/src/compression.rs`: buffer compression format

## Practical Summary

IPC record-batch encoding is best understood as:

- recursive schema-guided traversal,
- append buffers in canonical order,
- emit enough metadata for the reader to replay the same traversal in reverse.
