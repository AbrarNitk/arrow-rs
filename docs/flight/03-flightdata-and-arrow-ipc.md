# 3. `FlightData` and Arrow IPC

This chapter is the heart of Flight.

If you understand `FlightData`, you understand that Flight is not a new columnar format. It is a protocol for *streaming Arrow IPC messages through gRPC*.

## The Envelope

At the wire level, one streamed item is `FlightData`.

Conceptually:

```text
FlightData
├── flight_descriptor?   optional, usually on first outbound message
├── data_header          Arrow IPC flatbuffer Message bytes
├── app_metadata         app-defined side metadata
└── data_body            raw Arrow buffer payload bytes
```

The crucial rule is:

- `data_header` tells the receiver what kind of Arrow IPC message this is,
- `data_body` holds the actual buffers referenced by that header.

In practice the header may describe:

- `Schema`
- `DictionaryBatch`
- `RecordBatch`

The decoder in `arrow-flight/src/decode.rs` explicitly switches on Arrow IPC `MessageHeader` and treats anything else as a protocol error.

## Why Arrow IPC Is Embedded Instead of Rewritten

Reusing Arrow IPC gives Flight several benefits:

- one canonical schema model across files, shared memory, IPC, and network transport,
- one canonical batch layout,
- one canonical dictionary encoding model,
- shared encode/decode implementations with the rest of Arrow.

This is why `arrow-flight` leans on `arrow-ipc` instead of hand-serializing arrays. See:

- `arrow-flight/src/encode.rs`
- `arrow-flight/src/decode.rs`
- `arrow-flight/src/utils.rs`

## Stream Ordering Rules

A valid Flight batch stream is stateful.

Typical order:

```text
Schema
DictionaryBatch*
RecordBatch*
DictionaryBatch*
RecordBatch*
...
```

Important consequences:

- a `RecordBatch` cannot be decoded before its schema,
- dictionary batches update decoder state,
- multiple data messages may correspond to one logical input batch if the sender splits for gRPC size reasons.

The decoder in `FlightDataDecoder::extract_message` enforces exactly this kind of stream-state logic.

## Why the Schema Must Arrive First

Arrow buffers are not self-describing enough to recover the logical array structure on their own. The decoder must know:

- field order,
- data types,
- child array relationships,
- dictionary ids,
- nullability semantics.

That is why the schema message is effectively the stream contract.

In `arrow-flight/src/decode.rs`, receiving a `DictionaryBatch` or `RecordBatch` before a schema is a protocol error.

## Dictionary Messages: Why They Exist

Dictionary arrays split logical values into:

- keys in the batch body,
- dictionary values stored separately.

This is efficient because repeated values are factored out. But it means the stream is no longer stateless. The receiver must remember previously supplied dictionary values by id.

That is why `FlightDataDecoder` maintains:

```text
FlightStreamState
├── schema
└── dictionaries_by_field: HashMap<i64, ArrayRef>
```

in `arrow-flight/src/decode.rs`.

## The Most Important Consequence: Flight Streams Are Stateful

Many newcomers think each `FlightData` message is independently decodable. That is wrong.

For dictionary-using streams, decoding one batch depends on:

- the active schema,
- prior dictionary batches,
- the IPC message header in the current item,
- the body buffers in the current item.

This statefulness is why the crate exposes both:

- `FlightDataDecoder` for low-level message-aware decoding
- `FlightRecordBatchStream` for "give me batches" consumption

The higher-level wrapper hides internal schema and dictionary churn. The lower-level wrapper exposes it when you need protocol visibility.

## `app_metadata`: Why a Side Channel Exists

`app_metadata` is intentionally separate from Arrow IPC payload bytes.

That is useful because applications often need to associate:

- tracing ids,
- custom server hints,
- progress fragments,
- domain-specific metadata

with a stream item, without modifying the Arrow batch layout itself.

The tests in `arrow-flight/tests/encode_decode.rs` show this clearly: metadata can ride on the schema message while the underlying record batch stays normal Arrow IPC.

## Size Limits and Batch Splitting

gRPC deployments commonly impose message size limits. A single Arrow `RecordBatch` may be too large for one safe `FlightData` message.

The encoder therefore has a policy layer:

- estimate batch size,
- slice the batch zero-copy into smaller row ranges,
- encode each slice as a separate Flight payload.

This happens in `split_batch_for_grpc_response` inside `arrow-flight/src/encode.rs`.

Why this policy makes sense:

- preserve order,
- avoid forcing users to pre-split application batches,
- keep transport constraints local to the transport encoder,
- use Arrow slice semantics so splitting is cheap.

This is a good example of Flight respecting Arrow semantics while adapting them to network realities.

## The Data Plane Diagram

```text
RecordBatch stream
  |
  | encode via arrow-ipc
  v
FlightData stream
  |
  | item 1: Schema message
  | item 2: DictionaryBatch (optional)
  | item 3: RecordBatch
  | item 4: RecordBatch
  | ...
  v
gRPC stream
  |
  | decode header type
  | maintain schema + dictionary state
  v
RecordBatch stream
```

## Schema in `FlightInfo` vs Schema in `FlightData`

This often confuses readers.

The protocol can expose schema in two places:

- `FlightInfo.schema`: discovery-time schema
- first `FlightData` schema message: stream-time schema

Why both make sense:

- discovery needs schema before opening the stream,
- the actual stream still needs an in-band schema to be self-decodable.

This redundancy is not waste. It serves two different stages of the interaction.

## Source Markers

- `arrow-flight/src/arrow.flight.protocol.rs`: `FlightData`, `SchemaResult`, related docs
- `arrow-flight/src/encode.rs`: schema emission, batch splitting, dictionary handling
- `arrow-flight/src/decode.rs`: stream-state machine for schema/dictionaries/batches
- `arrow-flight/src/utils.rs`: utility conversions between `RecordBatch` and `FlightData`
- `arrow-flight/tests/encode_decode.rs`: roundtrip behavior, metadata, splitting, dictionary cases

## Implementation Reading Advice

When you read encode/decode code, always separate:

1. Arrow IPC semantics coming from `arrow-ipc`
2. Flight transport policy added by `arrow-flight`

That separation explains most of the crate design.
