# Arrow Flight Deep Dive

This tutorial set is for reading Apache Arrow Flight the way an implementer reads it: from transport constraints upward, with the wire model, stream state, and crate code all connected.

The goal is not "learn the API surface." The goal is stronger:

- understand why Flight exists when Arrow IPC already exists,
- understand why the protocol splits discovery from transfer,
- understand why `FlightData` carries Arrow IPC messages instead of inventing a second columnar wire format,
- understand how `arrow-flight` in this repository maps those decisions into generated protobuf types, tonic services, stream adapters, and high-level clients,
- understand enough to explain the protocol publicly or build a compatible implementation from scratch.

Read in order:

1. [01-why-flight-exists.md](./01-why-flight-exists.md)
2. [02-control-plane-types.md](./02-control-plane-types.md)
3. [03-flightdata-and-arrow-ipc.md](./03-flightdata-and-arrow-ipc.md)
4. [04-rpc-lifecycle-and-state-machines.md](./04-rpc-lifecycle-and-state-machines.md)
5. [05-encoding-and-decoding-in-arrow-rs.md](./05-encoding-and-decoding-in-arrow-rs.md)
6. [06-client-server-implementation.md](./06-client-server-implementation.md)
7. [07-flight-sql-extension.md](./07-flight-sql-extension.md)
8. [08-auth-headers-errors-and-operations.md](./08-auth-headers-errors-and-operations.md)
9. [09-usage-patterns-and-examples.md](./09-usage-patterns-and-examples.md)
10. [10-worked-example-query-and-ingest.md](./10-worked-example-query-and-ingest.md)

## Reading Model

Use this mental stack throughout the chapters:

```text
Application semantics
  "run this query", "upload this batch", "list datasets"

Flight control plane
  FlightDescriptor, FlightInfo, FlightEndpoint, Ticket, Action, PollInfo

Flight data plane
  stream<FlightData>

Arrow IPC payload
  Schema / DictionaryBatch / RecordBatch flatbuffer messages + data buffers

gRPC transport
  unary calls + bidirectional streams + headers + trailers + status

HTTP/2 transport properties
  multiplexing, backpressure, message framing, TLS, auth metadata
```

If you lose track of what a type is doing, first ask which layer it belongs to. Most Flight confusion comes from mixing these layers together.

## Main Source Markers

- `arrow-flight/src/lib.rs`
- `arrow-flight/src/arrow.flight.protocol.rs`
- `arrow-flight/src/client.rs`
- `arrow-flight/src/encode.rs`
- `arrow-flight/src/decode.rs`
- `arrow-flight/src/utils.rs`
- `arrow-flight/src/streams.rs`
- `arrow-flight/src/trailers.rs`
- `arrow-flight/src/error.rs`
- `arrow-flight/src/sql/mod.rs`
- `arrow-flight/src/sql/client.rs`
- `arrow-flight/src/sql/server.rs`
- `arrow-flight/examples/server.rs`
- `arrow-flight/examples/flight_sql_server.rs`
- `arrow-flight/tests/client.rs`
- `arrow-flight/tests/encode_decode.rs`

## How To Use These Docs

If you are new to Flight, read chapters 1 to 4 first.

If you need to implement a server, read 4, 5, 6, 8, and 9.

If you need to understand Flight SQL, do not skip the base protocol chapters. Flight SQL is not a separate transport. It is a disciplined reuse of the base protocol, mostly through `FlightDescriptor::cmd`, `Ticket`, `Action`, and `DoPut`.
