# 5. Encoding and Decoding in `arrow-flight`

This chapter is about the implementation mechanics in this repository: how `arrow-flight` turns Arrow values into Flight streams and back.

## Encoder Architecture

The main entry point is `FlightDataEncoderBuilder` in `arrow-flight/src/encode.rs`.

It captures transport policy:

- target max message size,
- IPC write options,
- application metadata,
- optional known schema,
- optional initial `FlightDescriptor`,
- dictionary handling mode.

That is already telling you something important about the crate design:

- Arrow layout policy stays in `arrow-ipc`,
- Flight transport policy is layered on top.

## Why a Builder Is the Right Shape

Encoding has multiple cross-cutting decisions:

- should schema be emitted immediately or inferred from first batch?
- should dictionaries be preserved or hydrated?
- how large should gRPC messages be?
- should app metadata be attached?

A builder makes these explicit without proliferating many constructor variants.

## The Encoder Stream Model

`FlightDataEncoder` itself is a `Stream<Item = Result<FlightData>>`.

That is the correct abstraction because:

- Flight servers need to stream,
- upstream batch production may itself fail mid-stream,
- the encoder may expand one input batch into multiple output messages.

Internal structure:

```text
FlightDataEncoder
├── inner stream<Result<RecordBatch>>
├── schema?
├── queue<VecDeque<FlightData>>
├── FlightIpcEncoder
├── descriptor?
├── app_metadata?
└── dictionary_handling
```

The queue exists because encoding one input batch may enqueue:

- schema message,
- zero or more dictionary messages,
- one or more record batch messages.

So polling the encoder is not a 1:1 transformation.

## Schema Emission Policy

If schema is known up front, the encoder can emit it immediately. Otherwise it waits for the first batch and derives schema from that batch.

Why this makes sense:

- no need to require empty-schema boilerplate when data exists,
- still supports protocols where a schema-only stream is useful,
- preserves the invariant that the first meaningful payload is schema.

The test `test_zero_batches_schema_specified` in `arrow-flight/tests/encode_decode.rs` shows why this matters: some consumers expect schema even with zero batches.

## Dictionary Handling Policy

This crate exposes two dictionary strategies:

- `Hydrate`
- `Resend`

### `Hydrate`

Convert dictionary arrays to their underlying value arrays before transport.

Benefits:

- simpler client compatibility,
- no need to maintain dictionary state,
- less protocol complexity.

Cost:

- potentially more bytes on the wire.

### `Resend`

Transmit dictionary data explicitly as dictionary messages.

Benefits:

- preserves dictionary encoding semantics,
- potentially more compact transport.

Costs:

- client/server must track dictionary ids,
- repeated dictionary sends may occur,
- more protocol-state complexity.

The code comments in `arrow-flight/src/encode.rs` are worth reading closely here. They explain that this implementation currently resends dictionary FlightData for batches that contain dictionaries, rather than maintaining a sophisticated delta protocol.

This is a pragmatic choice: correctness and interoperability first, richer dictionary transport later.

## Preparing Schema for Flight

Functions like `prepare_field_for_flight` and `prepare_schema_for_flight` recursively walk Arrow types before transport.

Why that recursion exists:

- nested types may contain dictionaries below the top level,
- view/list/struct/union/map forms must preserve logical shape,
- transport policy may need to rewrite field type representations.

This is a good example of how Flight transport cannot be fully "type blind." It must understand Arrow type structure well enough to preserve or transform transport-relevant features.

## Batch Splitting

`split_batch_for_grpc_response` estimates a batch's buffer memory size and slices the batch across row ranges.

Why row slicing instead of byte slicing:

- Arrow arrays are row-addressable at the logical level,
- arbitrary byte slicing would violate array boundaries,
- batch slices can stay zero-copy from the original arrays.

This is a strong design choice. It respects Arrow's structural invariants and still solves transport constraints.

## Decoder Architecture

The decoder side has two layers:

- `FlightDataDecoder`: low-level decoded message stream
- `FlightRecordBatchStream`: high-level record batch stream

`FlightDataDecoder` is the true protocol engine. It:

- parses `data_header` with `arrow_ipc::root_as_message`,
- checks the `MessageHeader`,
- updates schema state,
- updates dictionary state,
- reconstructs `RecordBatch` values.

`FlightRecordBatchStream` then filters that lower-level stream into user-facing batches and enforces simple invariants like "multiple schema messages in a single logical stream are unexpected."

## Why the Two-Layer Decoder Is Correct

One layer is not enough because advanced users sometimes need:

- direct access to `app_metadata`,
- visibility into schema messages,
- custom handling of low-level FlightData flow.

But many users just want `Stream<RecordBatch>`.

The crate correctly serves both.

## Stream Adapters for Error Propagation

`arrow-flight/src/streams.rs` contains two small but important adapters:

- `FallibleRequestStream`
- `FallibleTonicResponseStream`

These solve a non-obvious problem:

- tonic request streams are often infallible from its point of view,
- user-produced input streams may fail,
- those failures should not disappear.

So the crate intercepts client-side errors and forwards them through a oneshot channel into the returned response stream.

This is exactly the kind of engineering detail a rewrite-from-scratch implementation must get right.

## Trailers

`arrow-flight/src/trailers.rs` extracts gRPC trailers lazily once a streaming response is exhausted.

Why this is necessary:

- trailers are only known at stream completion,
- callers still want a clean stream interface while reading,
- metadata like final status or server info may matter after the last batch.

## Source Markers

- `arrow-flight/src/encode.rs`: encoder builder, queueing, schema policy, dictionary policy, batch splitting
- `arrow-flight/src/decode.rs`: low-level stream-state decoder and record-batch wrapper
- `arrow-flight/src/streams.rs`: error propagation adapters for streaming client calls
- `arrow-flight/src/trailers.rs`: end-of-stream trailer extraction
- `arrow-flight/tests/encode_decode.rs`: empty streams, dictionaries, metadata, batch splitting, schema handling

## Reimplementation Checklist

If you were rebuilding this crate, you would need:

1. a stateful Arrow IPC encoder over `RecordBatch` streams,
2. a stateful Arrow IPC decoder over `FlightData` streams,
3. explicit schema-first invariants,
4. dictionary-tracking policy,
5. transport-aware batch splitting,
6. mid-stream error propagation,
7. header and trailer capture.

That is the mechanical core of Flight support.
