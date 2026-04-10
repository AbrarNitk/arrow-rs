# 2. Control-Plane Types

Flight's control plane exists to answer a simple question with enough structure for real systems:

> What data should be transferred, where can it be fetched, and how should the client reason about that fetch?

The protobuf types in `arrow-flight/src/arrow.flight.protocol.rs` answer that question.

## `FlightDescriptor`: Naming a Logical Dataset

`FlightDescriptor` identifies *what the client wants*, not *how to fetch it yet*.

It has two modes:

- `PATH`: a stable, path-like dataset identifier
- `CMD`: an opaque command blob

This split is one of the most important protocol choices.

### Why `PATH` Exists

`PATH` is for datasets that behave like named resources:

- tables,
- materialized datasets,
- logical namespaces.

It is easy to reason about and cache because the identifier is stable and structured.

### Why `CMD` Exists

Many analytics requests are not stable named resources. They are generated computations:

- SQL query text,
- serialized logical plans,
- parameterized prepared statement handles,
- domain-specific commands.

That is why `CMD` is opaque bytes. The Flight base protocol refuses to overfit the meaning of "a request for data." It gives the application room to define it.

This is the same design move later used by Flight SQL, where SQL commands are serialized protobuf messages and placed into `FlightDescriptor::cmd`.

## `FlightInfo`: Discovery Result

`FlightInfo` is the answer to "what do I fetch?"

It contains:

- schema in IPC form,
- the original descriptor,
- one or more endpoints,
- estimated or exact totals,
- an `ordered` flag,
- optional application metadata.

Diagram:

```text
FlightInfo
├── schema                IPC-encoded Arrow schema
├── flight_descriptor     what logical request this describes
├── endpoint[]            where to fetch partitions/results
├── total_records         advisory count or -1
├── total_bytes           advisory size or -1
├── ordered               whether endpoint order is data order
└── app_metadata          app-specific discovery metadata
```

### Why Schema Is in `FlightInfo`

The client often needs the schema *before* opening any data stream:

- for planning memory,
- choosing decoders,
- presenting result shape,
- validating query expectations.

Putting schema in `FlightInfo` lets the client reason about the result before consuming data.

### Why Endpoints Are a List

A distributed server may want to split one logical result into several fetchable pieces:

- partitions on different workers,
- replicas near different clients,
- shard-local result streams.

That is why `FlightInfo` does not return just one ticket. It returns a list of `FlightEndpoint`s.

### Why `ordered` Exists

Partitioned execution and global ordering are in tension.

If a query result is partitioned, clients naturally want parallelism. But if row order matters, parallel fetch can break that order. `ordered` lets the server communicate when endpoint order itself is meaningful.

The protocol docs in the generated Rust code are careful here: clients *may ignore* this flag, so applications that require strict ordering should either enforce client behavior or return a single endpoint.

## `FlightEndpoint`: Fetch Coordinates

`FlightEndpoint` bridges discovery and transfer.

It typically contains:

- a `Ticket` used with `DoGet`
- zero or more `Location`s telling the client where this endpoint may be fetched
- endpoint-level application metadata
- optional expiration time

The important policy here is that a client does not fetch data "by descriptor." It fetches data "by ticket."

## Why `Ticket` Is Opaque

`Ticket` is intentionally opaque bytes.

This is a good decision because the server may encode:

- a partition identifier,
- a query handle,
- an authorization-bound token,
- a cache key,
- a serialized plan fragment.

If the protocol had standardized ticket internals, it would have frozen server architecture. Opaque tickets preserve implementation freedom.

## `PollInfo`: Long-Running Query State

`PollInfo` exists because not every query can produce complete endpoint information immediately.

It contains:

- an optional current `FlightInfo`,
- an optional next `FlightDescriptor` for polling again,
- optional progress,
- optional expiration time.

This is a subtle but powerful design.

### Why Polling Returns Another Descriptor

The next poll token is modeled as another `FlightDescriptor`, not a special side channel. That keeps the polling model inside the same descriptor abstraction: "here is the next logical handle for this evolving query."

### Why `PollInfo` Returns Complete `FlightInfo`

The generated docs note that each `PollInfo` contains a complete `FlightInfo`, not just a delta. That simplifies clients:

- they do not need complex merge logic for all fields,
- they can treat each poll result as a fresh snapshot,
- the server can append endpoints over time.

## Other Control Types

### `Criteria`

Used by `ListFlights`. It is opaque bytes because listing policy is application-defined.

### `Action` and `ActionType`

These exist because some operations are not dataset discovery and not data transfer. They are "do this management or control operation." Making them explicit prevents overloading `CMD` for everything.

### `SchemaResult`

Used by `GetSchema`. The result is IPC-encoded schema bytes, because Flight wants the base Arrow schema representation to stay canonical.

## Control-Plane Flow

```text
Client intent
  |
  | PATH or CMD
  v
FlightDescriptor
  |
  | GetFlightInfo / PollFlightInfo
  v
FlightInfo / PollInfo
  |
  | endpoint[i]
  v
FlightEndpoint
  |
  | ticket
  v
DoGet(ticket)
```

## How `arrow-flight` Exposes These Types

The crate re-exports the generated protobuf structs from `arrow-flight/src/lib.rs`, while also exposing convenience APIs:

- `FlightDescriptor::new_cmd(...)`
- `FlightDescriptor::new_path(...)` style constructors
- `FlightClient::get_flight_info`
- `FlightClient::poll_flight_info`
- `FlightClient::get_schema`

The design pattern here is useful:

- keep generated protocol structs available,
- layer ergonomics above them,
- avoid hiding the wire model from advanced users.

## Source Markers

- `arrow-flight/src/arrow.flight.protocol.rs`: `FlightDescriptor`, `FlightInfo`, `PollInfo`, `FlightEndpoint`, `Ticket`
- `arrow-flight/src/lib.rs`: re-exports and helper schema wrappers
- `arrow-flight/src/client.rs`: high-level control-plane client calls
- `arrow-flight/tests/client.rs`: request/response behavior for `get_flight_info`, `poll_flight_info`, `get_schema`

## What To Notice In Code

When reading the generated structs, pay attention to what is *not* specified:

- ticket contents,
- criteria expression semantics,
- app metadata semantics,
- command bytes semantics.

That looseness is deliberate. Flight standardizes transport structure and Arrow payloads, not the server's business model.
