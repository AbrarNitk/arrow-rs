# 1. Why Flight Exists

Arrow Flight exists because Arrow IPC alone solves the *representation* problem, not the *distributed transport* problem.

Arrow IPC gives you a compact, language-neutral way to describe:

- a schema,
- dictionaries,
- record batches,
- and the underlying Arrow buffers.

But a real distributed system still needs answers to different questions:

- How does a client discover what datasets exist?
- How does a client ask for "the result of this query" instead of "a file at this path"?
- How does a server split one logical result across multiple fetchable partitions?
- How do authentication, TLS, retries, streaming backpressure, and partial progress fit in?
- How do uploads, downloads, metadata actions, and long-running queries share one transport?

Flight is Arrow's answer to that second layer.

## The Design Pressure

A high-performance analytics transport wants all of the following at once:

```text
Need                                  Why it matters
-----------------------------------   ---------------------------------------
Columnar payloads                     avoid row re-materialization
Streaming                             don't buffer entire result sets
Bidirectional RPC                     uploads and exchanges need both directions
Endpoint discovery                    clients may fetch from multiple locations
Language neutrality                   servers and clients are often heterogeneous
Auth/TLS/metadata                     production systems need control hooks
Backpressure                          readers must naturally pace writers
```

If Flight invented a brand-new columnar wire format, it would duplicate Arrow IPC and split the ecosystem. If it reused only raw Arrow IPC files or streams, it would still be missing query discovery, endpoint routing, actions, and control messages.

So Flight composes:

```text
gRPC / HTTP2        -> transport, streaming, metadata, status, TLS
protobuf messages   -> control-plane types
Arrow IPC           -> columnar payload encoding
```

That composition is the central design decision.

## Why gRPC Makes Sense

Flight chose gRPC because the problem is not "serialize bytes." The problem is "move structured datasets and control messages over a production RPC transport."

gRPC contributes:

- streaming RPCs for `DoGet`, `DoPut`, and `DoExchange`,
- unary RPCs for discovery and metadata requests,
- HTTP/2 multiplexing so many logical streams can share one connection,
- built-in metadata for auth and request context,
- status and trailers for end-of-stream signaling,
- mature language support.

This is why the Rust crate has both generated tonic client/server code and higher-level Arrow-aware wrappers. The generated layer speaks gRPC. The higher layer understands Arrow stream semantics. See `arrow-flight/src/lib.rs` and the generated service modules in `arrow-flight/src/arrow.flight.protocol.rs`.

## Why Discovery Is Separate From Data Transfer

One of Flight's most important policy decisions is separating:

- *what exists / how to fetch it* from
- *the actual batch stream*

That is why `GetFlightInfo` and `PollFlightInfo` exist separately from `DoGet`.

This separation makes sense because the server may need to:

- plan a query,
- authenticate and authorize it,
- decide partitioning,
- choose different endpoints,
- return different tickets for different partitions,
- expose partial progress before data is complete.

If discovery and transfer were fused into one RPC, many distributed execution patterns would become awkward or impossible.

## Control Plane vs Data Plane

Keep this distinction sharp:

```text
Control plane
  FlightDescriptor
  FlightInfo
  FlightEndpoint
  Ticket
  PollInfo
  Action / ActionType

Data plane
  stream<FlightData>
    where FlightData embeds Arrow IPC schema/dictionary/record-batch messages
```

The control plane tells the client *what stream to open*. The data plane carries the Arrow bytes for that stream.

## Why `FlightData` Reuses Arrow IPC

`FlightData` is not a second Arrow format. It is a transport envelope around Arrow IPC message pieces:

- `data_header`: IPC flatbuffer `Message`
- `data_body`: raw buffer bytes referenced by that message
- `app_metadata`: application-defined side channel
- `flight_descriptor`: optional descriptor on the first outgoing message

This means Flight inherits Arrow IPC's physical layout rather than redefining schemas, batches, dictionaries, and buffers. In Rust, the encode/decode machinery is deliberately thin around Arrow IPC:

- encoding in `arrow-flight/src/encode.rs`
- decoding in `arrow-flight/src/decode.rs`
- conversion helpers in `arrow-flight/src/utils.rs`

That reuse is a major reason Flight is tractable.

## The Core Mental Picture

```text
Client
  |
  | 1. "Tell me what this dataset/query is"
  v
GetFlightInfo / PollFlightInfo
  |
  | <- FlightInfo { schema, endpoints[], descriptor, totals, metadata }
  |
  | 2. "Now send me endpoint N"
  v
DoGet(Ticket)
  |
  | <- FlightData(Schema)
  | <- FlightData(DictionaryBatch)?*
  | <- FlightData(RecordBatch)*
  v
Decoded Arrow RecordBatch stream
```

The same pattern generalizes:

- upload uses `DoPut`
- bidirectional pipelines use `DoExchange`
- management operations use `DoAction`
- discovery uses `ListFlights`, `GetSchema`, `ListActions`

## Why This Is a Good Fit for Analytics

Flight favors large, structured, streaming transfers.

That policy makes sense because analytics systems often:

- produce results incrementally,
- benefit from zero-copy or near-zero-copy columnar interchange,
- need separate query planning and data retrieval,
- care more about throughput and structured semantics than human readability.

It is not a universal RPC framework. It is a specialized transport for Arrow-shaped data systems.

## Source Markers

- `arrow-flight/src/lib.rs`: crate layering and public surface
- `arrow-flight/src/arrow.flight.protocol.rs`: generated protocol model
- `arrow-flight/src/client.rs`: high-level client that turns protocol calls into Arrow-aware operations
- `arrow-flight/src/encode.rs`: Arrow `RecordBatch` -> `FlightData`
- `arrow-flight/src/decode.rs`: `FlightData` -> Arrow `RecordBatch`

## Questions To Carry Forward

After this chapter, the next questions are:

1. How does the control plane identify datasets and endpoints?
2. What exactly is inside `FlightInfo`, `Ticket`, and `FlightEndpoint`?
3. How does a `FlightData` stream carry schema, dictionaries, and batches?

Those are the subjects of the next two chapters.
