# 4. RPC Lifecycle and State Machines

Flight is easiest to understand when each RPC is treated as a state transition in a dataset conversation.

## The RPC Set

The generated service in `arrow-flight/src/arrow.flight.protocol.rs` exposes a small set of verbs:

- `Handshake`
- `ListFlights`
- `GetFlightInfo`
- `PollFlightInfo`
- `GetSchema`
- `DoGet`
- `DoPut`
- `DoExchange`
- `DoAction`
- `ListActions`

These are not redundant. Each exists because analytics transports need a different dataflow shape.

## `Handshake`

Purpose:

- establish authentication/session state
- possibly return a token

Why separate from every other request:

- keeps auth bootstrapping explicit,
- avoids forcing every request to invent its own login envelope,
- works with gRPC metadata and server session models.

In Rust, `FlightClient::handshake` in `arrow-flight/src/client.rs` sends a one-item request stream and expects exactly one response payload.

That "exactly one" check is important. It turns a loosely typed stream RPC into a stronger local contract.

## `ListFlights`

Purpose:

- enumerate available flights according to application-specific criteria.

This is discovery over collections, not transfer of one chosen dataset.

## `GetFlightInfo`

Purpose:

- resolve a descriptor into fetch coordinates.

Think of it as planning:

```text
descriptor -> server planning / authorization / partitioning -> FlightInfo
```

Why it is unary:

- the result is a metadata object,
- the client usually wants the whole answer before choosing endpoints.

## `PollFlightInfo`

Purpose:

- support long-running planning or incremental endpoint publication.

This is essential for systems where:

- query planning is expensive,
- execution starts before all partitions are known,
- partial results can be consumed while the overall job is still progressing.

Diagram:

```text
client submits descriptor
  |
  v
PollFlightInfo(desc0)
  -> PollInfo { info = partial, flight_descriptor = desc1, progress = 0.3 }
PollFlightInfo(desc1)
  -> PollInfo { info = larger partial, flight_descriptor = desc2, progress = 0.7 }
PollFlightInfo(desc2)
  -> PollInfo { info = complete, flight_descriptor = None, progress = 1.0 }
```

This is cleaner than blocking `GetFlightInfo` for long periods or inventing an external job-status API.

## `GetSchema`

Purpose:

- ask for schema only.

Why it exists even though `FlightInfo` includes schema:

- schema discovery can be cheaper than full planning,
- some clients only need result shape,
- it avoids endpoint planning work when data transfer is not imminent.

## `DoGet`

Purpose:

- consume a stream identified by `Ticket`.

This is the core read path:

```text
Ticket -> DoGet -> stream<FlightData> -> decoded RecordBatch stream
```

Why ticket-based fetch makes sense:

- tickets can bind authorization and partition identity,
- discovery can hand out multiple fetch handles,
- the server stays free to change internal request representation.

## `DoPut`

Purpose:

- client uploads a stream of `FlightData`
- server replies with a stream of `PutResult`

Why the response is also a stream:

- uploads may be acknowledged incrementally,
- the server may produce application metadata during ingestion,
- streaming symmetry avoids forcing full upload completion before any response.

In the Rust client, `do_put` accepts a stream of `Result<FlightData>` and uses adapters in `arrow-flight/src/streams.rs` to propagate client-side stream errors back into the returned response stream.

That is a subtle but important mechanical decision: the API preserves streaming and still surfaces local producer failures.

## `DoExchange`

Purpose:

- bidirectional `FlightData` stream where both sides can speak during the session.

This is for cases where `DoGet` and `DoPut` are too one-sided:

- interactive compute exchange,
- pushdown protocols,
- stateful transforms,
- mixed control/data conversations over one stream.

Why Flight keeps this separate from `DoPut`:

- it advertises stronger semantics,
- implementers can support upload-only without full duplex exchange logic,
- callers know both sides may produce data concurrently.

## `DoAction` and `ListActions`

Purpose:

- out-of-band management and control operations.

Typical use:

- cancel work,
- create prepared statements,
- transaction control,
- server-specific maintenance hooks.

Actions prevent users from smuggling every management operation through fake descriptors or tickets.

## State Machines

### Read State Machine

```text
Descriptor
  -> GetFlightInfo / PollFlightInfo
  -> FlightInfo
  -> choose endpoint
  -> DoGet(ticket)
  -> Schema
  -> Dictionaries? / RecordBatches*
  -> EOF + trailers
```

### Write State Machine

```text
Client batches
  -> encode to FlightData
  -> DoPut(stream)
  -> server consumes schema/dictionaries/batches
  -> PutResult*
  -> final status / trailers
```

### Exchange State Machine

```text
Client outbound FlightData*  <====>  Server outbound FlightData*
each side maintains its own decode state
```

That last line matters: duplex exchange means each side may need both an encoder and decoder state machine at once.

## Source Markers

- `arrow-flight/src/arrow.flight.protocol.rs`: RPC service shape and message docs
- `arrow-flight/src/client.rs`: high-level methods for each RPC
- `arrow-flight/src/streams.rs`: client-side stream error adaptation for `DoPut` and `DoExchange`
- `arrow-flight/src/trailers.rs`: deferred trailer extraction after stream exhaustion
- `arrow-flight/tests/client.rs`: behavior of handshake, discovery, `DoGet`, and metadata flow

## Design Summary

Flight's RPC set is good because each method corresponds to a distinct distributed-system need:

- discovery,
- fetch,
- upload,
- duplex processing,
- long-running planning,
- management.

The protocol stays small because it standardizes shapes of interaction, not every possible application command.
