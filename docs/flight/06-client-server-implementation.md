# 6. Client and Server Implementation

This chapter connects the protocol model to the Rust API design in this repository.

## Three Layers in the Crate

The crate surface in `arrow-flight/src/lib.rs` naturally separates into three levels:

```text
Level 1: generated protocol layer
  prost message structs
  tonic client/server service bindings

Level 2: Arrow-aware transport helpers
  encode, decode, utils, trailers, streams, error

Level 3: ergonomic clients and service helpers
  FlightClient
  FlightSqlServiceClient
  FlightSqlService
```

This layering is exactly what you want in a systems crate:

- generated code stays close to the spec,
- reusable helpers capture transport mechanics,
- ergonomic APIs help common use without hiding the wire model.

## `FlightClient`

`FlightClient` in `arrow-flight/src/client.rs` is intentionally described in code as a "mid level" client.

That wording matters.

It is not the raw tonic client.
It is not an ORM-like abstraction either.
It is a convenience layer that:

- attaches request metadata,
- exposes Arrow-aware `do_get` and `do_exchange`,
- keeps access to the underlying generated `FlightServiceClient`.

This is a strong design choice because advanced users often need direct gRPC access for:

- custom interceptors,
- raw headers,
- nonstandard request shaping.

The crate does not trap those users behind an over-abstracted client.

## Metadata Injection

`FlightClient` stores a `MetadataMap` and applies it to outgoing requests.

Why this belongs in the mid-level client:

- auth headers are common,
- tracing and tenant headers are common,
- forcing callers to mutate every tonic request manually would be repetitive.

But the client still exposes `inner`, `inner_mut`, and `into_inner`, which preserves escape hatches.

## `DoGet` Return Shape

`FlightClient::do_get` returns `FlightRecordBatchStream`, not raw tonic `Streaming<FlightData>`.

Why this is the right default:

- most callers want decoded Arrow batches,
- schema/dictionary state handling is nontrivial,
- headers and trailers are still preserved on the wrapper.

This design says: "default to the useful abstraction, but do not destroy transport visibility."

## `DoPut` and `DoExchange`

The client methods for upload and duplex use the stream adapters from `streams.rs`.

This is a key implementation detail:

- request-side failures are captured,
- the server response stream still propagates them,
- the API stays streaming.

Without that adaptation, a client-side producer error could terminate the request silently or force callers into awkward buffering.

## Server Side: Base Flight

The example in `arrow-flight/examples/server.rs` shows the generated `FlightService` trait from tonic.

Each method has an associated response stream type, for example:

- `DoGetStream = BoxStream<'static, Result<FlightData, Status>>`
- `DoPutStream = BoxStream<'static, Result<PutResult, Status>>`

This tells you something fundamental about Flight servers:

- the server owns stream production,
- backpressure is exercised by stream polling,
- the protocol is naturally async and incremental.

## Why the Base Server Example Is Sparse

The example methods are mostly `unimplemented!`.

That is not because the crate lacks direction. It is because the base service trait is intentionally low-level. The server author must define:

- how descriptors map to datasets,
- what tickets mean,
- how auth works,
- what actions are supported,
- how batches are produced or consumed.

That policy cannot live in the generic crate.

## Utility Helpers

`arrow-flight/src/utils.rs` contains convenience helpers:

- `batches_to_flight_data`
- `flight_data_to_batches`
- `flight_data_to_arrow_batch`

These are useful for examples, tests, or simple servers. But they should not be mistaken for the full transport engine. The deeper protocol-aware path still lives in the encoder and decoder modules.

## What a Production Server Usually Needs

A real server implementation adds policy around the protocol:

```text
descriptor parsing
  -> authorization
  -> logical planning
  -> endpoint selection
  -> ticket generation
  -> stream execution
  -> metrics / tracing / cancellation
```

Flight standardizes the transport boundaries around that policy.

## Why the Crate Exposes Both Raw and Ergonomic Layers

This is one of the best parts of the design.

If the crate exposed only raw generated protobuf/tonic code:

- common callers would repeatedly rewrite the same batch decode logic.

If it exposed only high-level batch APIs:

- advanced users would lose protocol control.

By exposing both, `arrow-flight` remains suitable for:

- application code,
- middleware authors,
- transport implementers,
- spec readers.

## Source Markers

- `arrow-flight/src/lib.rs`: crate public surface
- `arrow-flight/src/client.rs`: `FlightClient` API shape and request metadata behavior
- `arrow-flight/examples/server.rs`: tonic service implementation template
- `arrow-flight/src/utils.rs`: convenience conversions
- `arrow-flight/tests/client.rs`: client behavior, metadata, headers, trailers

## Reading Advice

When reading this crate, do not ask "where is the one server abstraction?"

Instead ask:

- which layer is generated protocol glue?
- which layer is Arrow transport logic?
- which layer is convenience for application authors?

Once you read it that way, the module boundaries make sense.
