# 3. Stream vs File Framing

Arrow IPC has one message model but two outer containers:

- stream format
- file format

This chapter explains why both exist and how `arrow-ipc` implements the difference.

## The Shared Inner Unit

Both variants ultimately write a sequence of encoded messages:

- schema message,
- dictionary messages,
- record batch messages,
- end marker.

The difference is not in the inner message type. The difference is in *how those messages are packaged for access*.

## Streaming Format

The stream format is optimized for sequential consumption.

Typical shape:

```text
Schema message
DictionaryBatch*
RecordBatch*
EOS marker
```

There is no footer and no random-access index.

### Why This Makes Sense

In a stream:

- the writer may not know the future,
- batches arrive incrementally,
- the consumer only needs the next message boundary.

Anything footer-like would require going back and rewriting the past, which is impossible or awkward on pipes and sockets.

### How `arrow-ipc` Writes It

`StreamWriter::try_new_with_options` in `arrow-ipc/src/writer.rs` writes the schema immediately.

Each call to `write` emits:

- any needed dictionary messages,
- then the record batch message.

`finish` writes an end-of-stream continuation with length `0`.

## File Format

The file format is optimized for random access.

High-level shape:

```text
ARROW_MAGIC
padding
Schema message
Dictionary blocks
Record batch blocks
EOS marker
Footer
footer length (i32 little endian)
ARROW_MAGIC
```

The exact payload is built in `FileWriter` in `arrow-ipc/src/writer.rs`.

### Why the Footer Exists

Random access requires an index.

The reader must know:

- where each dictionary block is,
- where each record batch block is,
- the schema and metadata for the whole file.

That is why the writer accumulates:

- `dictionary_blocks`
- `record_blocks`
- `block_offsets`

and only serializes the footer at the end.

### Why the Magic Bytes Appear at Both Ends

`ARROW_MAGIC` exists to make the file format self-identifying and to support footer discovery from the tail.

The trailer layout is:

```text
[4-byte footer length][6-byte ARROW_MAGIC]
```

The reader can seek to the end, read 10 bytes, validate the magic, recover footer length, then seek backwards to the footer.

This is implemented by `read_footer_length` and `FileReaderBuilder::build` in `arrow-ipc/src/reader.rs`.

That is the same general design logic as many random-access file formats: make the tail sufficient to discover the index.

## Framing Prefixes and Continuation Marker

`write_message` and `write_continuation` in `arrow-ipc/src/writer.rs` handle the prefix before each message.

The newer format uses:

```text
4 bytes: continuation marker = 0xFF FF FF FF
4 bytes: metadata length
```

followed by the flatbuffer metadata and then the body.

`CONTINUATION_MARKER` is defined in `arrow-ipc/src/lib.rs`.

### Why a Continuation Marker Exists

It makes the framing unambiguous and preserves compatibility around older protocol variants.

`try_schema_from_ipc_buffer` in `arrow-ipc/src/convert.rs` explicitly handles both:

- older 4-byte-length-only layout
- newer continuation-marker-prefixed layout

This is a good example of the crate carrying protocol history at the format boundary instead of spreading it everywhere.

## Stream vs File in One Diagram

```text
STREAM
  [msg][msg][msg][0-length EOS]

FILE
  [magic][msg][msg][msg][0-length EOS][footer][footer_len][magic]
```

The stream tells you "read next."
The file tells you "seek to block N."

## Random Access Tradeoff

The file format pays an extra cost:

- the writer must track block offsets,
- the file can only be finished once the footer is written.

But it gains:

- `FileReader::set_index`
- up-front knowledge of batch locations,
- easy mmap-style access patterns.

This is why the split into `StreamWriter`/`StreamReader` and `FileWriter`/`FileReader` is exactly the right API.

## Source Markers

- `arrow-ipc/src/lib.rs`: `ARROW_MAGIC`, `CONTINUATION_MARKER`
- `arrow-ipc/src/writer.rs`: `FileWriter`, `StreamWriter`, `write_message`, `write_continuation`
- `arrow-ipc/src/reader.rs`: `read_footer_length`, `FileReaderBuilder`, `FileReader`, `StreamReader`
- `arrow-ipc/src/convert.rs`: compatibility handling in `try_schema_from_ipc_buffer`

## Practical Summary

Use stream format when you are writing once and reading sequentially.

Use file format when you need seekable storage and random access to record batches.
