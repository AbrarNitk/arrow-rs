# 7. Flight SQL as an Extension Layer

Flight SQL is best understood as a disciplined reuse of Flight, not a separate system.

That point matters because many design choices in the code only make sense once you see the extension pattern clearly.

## The Core Policy

Flight SQL does **not** replace:

- `FlightDescriptor`
- `Ticket`
- `Action`
- `DoPut`
- `DoGet`

It uses them.

Specifically:

- command messages are serialized and stored in `FlightDescriptor::cmd`,
- tickets may carry serialized SQL-related ticket messages,
- actions carry transaction and prepared-statement control operations,
- `DoPut` is reused for update/ingest flows.

This is a very good protocol decision because it preserves one transport substrate for multiple higher-level services.

## What `arrow-flight` Adds

The `sql` module in `arrow-flight/src/sql/mod.rs` contains:

- generated protobuf messages for Flight SQL,
- `Any` helpers for typed packing/unpacking,
- `Command` enum for decoding command kinds,
- `FlightSqlServiceClient`,
- `FlightSqlService` helper trait,
- metadata helpers.

The comments in `sql/mod.rs` tell the story directly: Flight SQL is built on top of Arrow Flight RPC by encoding specific protobuf messages into base Flight fields.

## Why `Any` Is Used

Protobuf messages are not self-describing on their own. If `cmd` is just bytes, the receiver needs a typed wrapper to know what command those bytes represent.

The `Any` mechanism solves that by pairing:

- `type_url`
- serialized bytes

This lets the SQL layer multiplex many command types through one opaque `cmd` field without losing type identity.

That is why `sql/mod.rs` defines:

- `ProstMessageExt`
- `Any`
- `Command`

## `FlightSqlServiceClient`

The client in `arrow-flight/src/sql/client.rs` is a good case study in extension design.

It keeps:

- token handling,
- custom headers,
- underlying `FlightServiceClient`,

and adds SQL-specific convenience methods like:

- `execute`
- `execute_update`
- `execute_ingest`
- catalog/schema/table metadata calls

But the transport underneath is still base Flight:

- `execute` -> `GetFlightInfo` with a SQL command encoded into `cmd`
- `do_get` -> same base `DoGet`
- `execute_update` and ingest paths -> `DoPut`

## Why Updates Use `DoPut`

This is a subtle but elegant choice.

Updates and ingest are naturally write-oriented operations, and some of them need to stream input batches. Reusing `DoPut` means Flight SQL gets:

- streaming uploads,
- optional server acknowledgements,
- existing Arrow batch transport,
- no separate SQL-specific upload transport.

In `execute_ingest`, the client uses `FlightDataEncoderBuilder` and the same `FallibleRequestStream` pattern as the base client. That reuse proves the layering is sound.

## `FlightSqlService`

The helper trait in `arrow-flight/src/sql/server.rs` maps SQL command categories onto base Flight methods.

It provides typed hooks such as:

- `get_flight_info_statement`
- `do_get_statement`
- `get_flight_info_catalogs`
- transaction and prepared-statement action handlers

Why this helper exists:

- a raw `FlightService` would force every SQL server to decode `Any`, inspect command type URLs, and dispatch manually,
- the helper trait centralizes that dispatch logic,
- implementers can focus on SQL semantics instead of wire demultiplexing.

This is a classic "framework on top of protocol" layer.

## Server Example

`arrow-flight/examples/flight_sql_server.rs` is especially useful because it shows:

- basic auth in handshake,
- bearer token propagation,
- fake query results encoded as Arrow batches,
- prepared statement handling,
- SQL metadata responses,
- TLS configuration.

Treat it as a protocol map, not only as sample code.

## Diagram: Flight SQL Reuse

```text
Flight SQL API
  |
  | encode typed SQL command/action messages
  v
Base Flight fields
  FlightDescriptor::cmd
  Ticket
  Action.body
  DoPut / DoGet / DoAction
  |
  v
Arrow Flight transport
  |
  v
Arrow IPC batch payloads
```

## Why This Design Scales

Because the extension lives above Flight rather than beside it:

- base Flight infrastructure can be reused,
- auth/TLS/metadata behaviors carry over,
- generic Flight proxies or middleware can still operate,
- implementation effort is reduced.

If Flight SQL had created a separate transport, the ecosystem would fragment badly.

## Source Markers

- `arrow-flight/src/sql/mod.rs`: extension architecture and typed command support
- `arrow-flight/src/sql/client.rs`: SQL client on top of base Flight transport
- `arrow-flight/src/sql/server.rs`: helper trait that dispatches base Flight requests into SQL semantics
- `arrow-flight/examples/flight_sql_server.rs`: end-to-end example including auth, metadata, and SQL result handling

## Practical Lesson

If you are designing your own domain-specific service over Flight, Flight SQL is the model to study:

- keep transport generic,
- encode your control semantics into base Flight extension points,
- reuse `DoGet`/`DoPut`/`DoAction` instead of inventing a parallel protocol.
