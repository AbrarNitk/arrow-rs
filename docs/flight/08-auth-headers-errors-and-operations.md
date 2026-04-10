# 8. Auth, Headers, Errors, and Operations

A protocol is not production-ready because it can move data. It is production-ready when it can also handle identity, failure, lifecycle, and observability.

Flight relies heavily on gRPC for those concerns, and `arrow-flight` exposes that integration rather than hiding it.

## Request Metadata

Both base and SQL clients store request headers and attach them to outgoing requests:

- `FlightClient` uses a `MetadataMap`
- `FlightSqlServiceClient` stores headers and optional bearer token

Why metadata matters:

- auth tokens,
- tenant routing,
- tracing context,
- request correlation,
- domain-specific flags.

This is why client APIs expose metadata mutators instead of pretending all context belongs in Arrow batches.

## Handshake and Token Bootstrapping

The base `Handshake` RPC allows authentication payload exchange. The Flight SQL example server in `arrow-flight/examples/flight_sql_server.rs` demonstrates a realistic pattern:

```text
client sends Basic auth in metadata
  -> server validates credentials
  -> server returns bearer token in response metadata
  -> client stores token for subsequent calls
```

This pattern makes sense because:

- credentials are needed before normal data RPCs,
- tokens belong in transport metadata more than in batch payloads,
- subsequent requests become lighter-weight.

## TLS

The crate's Cargo features include multiple TLS options. That alone reflects an operational policy:

- Flight must work in real deployment environments,
- cryptographic backend choice can vary by platform or policy,
- transport security is not optional in many installations.

The Flight SQL example also configures `ServerTlsConfig`, reinforcing that Flight is designed for network service deployment, not only localhost demos.

## Headers vs `app_metadata`

This distinction is worth being precise about:

- gRPC headers/trailers are transport/session metadata
- `FlightData.app_metadata` is per-message application metadata

Use headers when the metadata belongs to the RPC context.
Use `app_metadata` when the metadata belongs to a specific streamed data item.

Confusing these two produces fragile designs.

## Errors

`arrow-flight/src/error.rs` defines `FlightError` with variants for:

- Arrow errors,
- tonic status,
- protocol violations,
- decode problems,
- external errors,
- not-yet-implemented cases.

This taxonomy matters because not all failures are alike.

### Why Protocol Errors Are Separate

A protocol error means "the remote side or local stream violated the expected Flight state machine." Examples:

- `RecordBatch` before schema,
- unexpected second handshake response,
- unsupported IPC message kind.

These are not normal business errors. They are correctness failures in the conversation itself.

### Why Tonic Status Is Preserved

If the remote server returns a proper gRPC status, the client should retain that identity. Wrapping and erasing it would lose important operational meaning like:

- unauthenticated,
- unavailable,
- internal,
- cancelled.

## Trailers and End-of-Stream Semantics

`arrow-flight/src/trailers.rs` captures trailers only when the stream is exhausted.

This is not incidental. In gRPC, trailers are end-of-stream metadata. That makes them a natural place for:

- final status details,
- audit metadata,
- metrics,
- server timing information.

The `FlightRecordBatchStream::trailers()` API explicitly reflects this lifecycle: do not expect trailers before the stream completes.

## Cancellation and Actions

The protocol includes cancel and renew-style operations modeled as actions, and the client wraps examples like `cancel_flight_info` and `renew_flight_endpoint` in `arrow-flight/src/client.rs`.

Why actions are used here instead of new RPC methods for every operational control:

- keeps the base RPC surface smaller,
- allows control operations to evolve,
- still standardizes the transport path.

This is a good compromise between strict standardization and protocol flexibility.

## Operational Diagram

```text
headers / auth metadata
  -> Handshake / normal RPC
  -> response headers
  -> streamed data
  -> final trailers
  -> status / cancellation semantics
```

This full lifecycle is what an operator sees in production, not just "a batch came back."

## Source Markers

- `arrow-flight/src/client.rs`: metadata attachment, handshake behavior, cancel/renew helpers
- `arrow-flight/src/trailers.rs`: lazy trailer extraction
- `arrow-flight/src/error.rs`: error taxonomy and conversions
- `arrow-flight/examples/flight_sql_server.rs`: token flow and TLS server configuration
- `arrow-flight/tests/client.rs`: headers and trailers asserted in `do_get`

## Implementation Lesson

A Flight implementation that only handles `DoGet` bytes but ignores:

- auth,
- headers,
- trailers,
- status mapping,
- cancellation

is not complete. It is only a demo transport.
