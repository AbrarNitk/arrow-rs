# 9. Usage Patterns and Worked Examples

This final chapter connects the mechanics to concrete use.

## Pattern 1: Sequential Streaming

Use `StreamWriter` and `StreamReader` when:

- data is produced incrementally,
- the consumer reads in order,
- random access is unnecessary.

Example shape:

```rust
use arrow_array::record_batch;
use arrow_ipc::reader::StreamReader;
use arrow_ipc::writer::StreamWriter;

let batch = record_batch!(("a", Int32, [1, 2, 3]))?;

let mut bytes = vec![];
{
    let mut writer = StreamWriter::try_new(&mut bytes, &batch.schema())?;
    writer.write(&batch)?;
    writer.finish()?;
}

let reader = StreamReader::try_new(bytes.as_slice(), None)?;
let batches: Result<Vec<_>, _> = reader.collect();
```

Why this is the right abstraction:

- no footer required,
- schema and dictionaries appear naturally before batches,
- works for pipes, sockets, and in-memory streams.

## Pattern 2: Seekable File Access

Use `FileWriter` and `FileReader` when:

- you want random batch access,
- you have a seekable destination/source,
- footer indexing is valuable.

Example shape:

```rust
use arrow_array::record_batch;
use arrow_ipc::reader::FileReader;
use arrow_ipc::writer::FileWriter;
use std::io::Cursor;

let batch = record_batch!(("a", Int32, [1, 2, 3]))?;

let mut file = vec![];
{
    let mut writer = FileWriter::try_new(&mut file, &batch.schema())?;
    writer.write(&batch)?;
    writer.write(&batch)?;
    writer.finish()?;
}

let mut reader = FileReader::try_new(Cursor::new(file), None)?;
reader.set_index(1)?;
let second = reader.next().unwrap()?;
```

## Pattern 3: Low-Level Schema Conversion

Use `IpcSchemaEncoder` and `fb_to_schema` when you need direct schema/flatbuffer handling without full batch IO.

This is useful for:

- embedding schemas elsewhere,
- implementing custom transport envelopes,
- testing or introspecting schema translation behavior.

Source marker:

- `arrow-ipc/src/convert.rs`

## Pattern 4: Push-Based Incremental Decode

Use `StreamDecoder` from `arrow-ipc/src/reader/stream.rs` when bytes arrive in arbitrary chunks.

This is especially useful for:

- object storage chunk iterators,
- networking stacks that do not yield message-aligned buffers,
- async adapters that feed byte slabs progressively.

The push-based decoder is one of the most important "systems integration" APIs in the crate.

## Pattern 5: Delta Dictionaries

Use `IpcWriteOptions::with_dictionary_handling(DictionaryHandling::Delta)` when:

- dictionary-encoded columns evolve incrementally across streamed batches,
- stable key assignments are preserved,
- you want smaller dictionary traffic.

Study:

- `arrow-ipc/tests/test_delta_dictionary.rs`
- stream-writer delta dictionary example in `arrow-ipc/src/writer.rs`

## Worked Example: One Streamed Batch

For a simple primitive batch, the stream conversation is:

```text
writer construction
  -> emit Schema message

writer.write(batch)
  -> emit RecordBatch message

writer.finish()
  -> emit EOS continuation

reader construction
  -> require first message to be Schema

reader.next()
  -> parse next message as RecordBatch
  -> decode nodes + buffers into arrays

reader.next()
  -> EOF
```

## Worked Example: One File with Two Batches

```text
FileWriter::try_new
  -> write header magic + schema message

write(batch0)
  -> write batch bytes
  -> record block offset/length

write(batch1)
  -> write batch bytes
  -> record block offset/length

finish()
  -> write EOS
  -> write footer containing block index
  -> write footer length
  -> write trailing magic

FileReader::try_new
  -> seek tail
  -> parse footer
  -> load dictionaries
  -> keep record blocks for random access
```

That is the full file lifecycle in one view.

## Worked Example: Flight Integration

Flight reuses IPC by taking the same schema/dictionary/record-batch message bytes and embedding them in `FlightData`.

That means:

- understanding IPC metadata/body layout is prerequisite to understanding Flight payloads,
- `arrow-flight` encode/decode logic is thin because `arrow-ipc` already does the hard work.

See:

- `docs/flight/03-flightdata-and-arrow-ipc.md`
- `arrow-flight/src/encode.rs`
- `arrow-flight/src/decode.rs`

## Source Markers

- `arrow-ipc/src/writer.rs`: file/stream usage APIs and delta dictionary example
- `arrow-ipc/src/reader.rs`: high-level file/stream readers
- `arrow-ipc/src/reader/stream.rs`: push-based stream decoder
- `arrow-ipc/src/convert.rs`: low-level schema roundtrip
- `arrow-ipc/tests/test_delta_dictionary.rs`: dictionary evolution behavior

## Final Summary

If you want the shortest correct description of Arrow IPC, use this:

> Arrow IPC is the canonical framed serialization of Arrow schemas, dictionaries, and record batches, using flatbuffer metadata plus ordered Arrow body buffers, packaged either as a sequential stream or as a random-access file.

That sentence is what the entire `arrow-ipc` crate is implementing.
