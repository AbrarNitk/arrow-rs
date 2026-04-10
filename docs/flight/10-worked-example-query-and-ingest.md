# 10. Worked Example: Query and Ingest End to End

This chapter traces two complete stories:

1. query result fetch
2. ingest/upload

The point is to connect control plane, data plane, and crate code in one place.

## Story A: Query Result Fetch

Assume a client wants the result of:

```text
SELECT id, name FROM users WHERE active = true
```

### Step 1: Represent the Logical Request

At the base Flight level, this becomes a `FlightDescriptor`.

If using Flight SQL, the SQL command is encoded as a protobuf message and packed into `FlightDescriptor::cmd`. See:

- `arrow-flight/src/sql/mod.rs`
- `arrow-flight/src/sql/client.rs`

Conceptually:

```text
CommandStatementQuery
  -> protobuf encode
  -> Any/type_url wrapper
  -> bytes into FlightDescriptor::cmd
```

### Step 2: Ask for `FlightInfo`

The client calls `GetFlightInfo`.

The server now performs server-specific work:

- authenticate,
- authorize,
- parse query,
- plan execution,
- decide partitioning,
- create tickets,
- encode the result schema.

Possible `FlightInfo` shape:

```text
FlightInfo
├── schema = Arrow schema for { id: Int64, name: Utf8 }
├── endpoint[0] = ticket for partition A
├── endpoint[1] = ticket for partition B
├── total_records = 12_000_000
├── total_bytes = -1
└── ordered = false
```

Why this makes sense:

- client can inspect schema immediately,
- result can be fetched in parallel,
- server keeps ticket internals private.

### Step 3: Open One Endpoint with `DoGet`

The client chooses one endpoint and calls `DoGet(ticket)`.

The server returns `stream<FlightData>`.

First items look like:

```text
message 1: Schema
message 2: RecordBatch rows 0..65535
message 3: RecordBatch rows 65535..131070
...
```

If dictionary arrays are present and transport preserves them:

```text
message 1: Schema
message 2: DictionaryBatch
message 3: RecordBatch
...
```

### Step 4: Decoder Reconstructs Batches

Inside `FlightRecordBatchStream` and `FlightDataDecoder`:

```text
read FlightData
  -> parse IPC message header
  -> if Schema: set stream schema, clear dictionaries
  -> if DictionaryBatch: update dictionary map
  -> if RecordBatch: reconstruct RecordBatch from schema + dictionaries + body buffers
```

That state machine is implemented in `arrow-flight/src/decode.rs`.

### Step 5: Stream Completion

Once the gRPC stream ends:

- the client may read trailers,
- the endpoint is complete,
- other endpoints may still be read independently.

This is where `arrow-flight/src/trailers.rs` becomes relevant.

## Story B: Ingest with `DoPut`

Assume the client wants to upload a `RecordBatch` stream into the server.

### Step 1: Build Application Batches

Application constructs Arrow arrays and `RecordBatch` values locally.

### Step 2: Encode to Flight

`FlightDataEncoderBuilder` transforms:

```text
stream<Result<RecordBatch>>
  ->
stream<Result<FlightData>>
```

Possible output:

```text
message 1: Schema
message 2: RecordBatch rows 0..49999
message 3: RecordBatch rows 50000..99999
```

If batches are large, `split_batch_for_grpc_response` may slice them.

### Step 3: Optional Descriptor

For ingest-like flows, the first message may carry a `FlightDescriptor` describing the upload target or command.

This behavior is supported by `with_flight_descriptor` in `arrow-flight/src/encode.rs`.

### Step 4: Server Consumes the Stream

On the server side, the implementation reads incoming `FlightData` and typically:

- decodes schema,
- decodes dictionaries,
- reconstructs record batches,
- writes them into storage or execution engine state.

The exact server policy is application-defined, but the transport mechanics are the same.

### Step 5: Server Sends `PutResult`

The server may acknowledge progress or final result metadata with `PutResult`.

Flight SQL update and ingest flows build on this same mechanism in `arrow-flight/src/sql/client.rs`.

## Full Diagram

```text
QUERY

client logical request
  -> FlightDescriptor
  -> GetFlightInfo
  -> FlightInfo(schema + endpoints)
  -> DoGet(ticket)
  -> FlightData(Schema)
  -> FlightData(DictionaryBatch)?
  -> FlightData(RecordBatch)*
  -> FlightRecordBatchStream


INGEST

client RecordBatch stream
  -> FlightDataEncoderBuilder
  -> FlightData(Schema)
  -> FlightData(DictionaryBatch)?
  -> FlightData(RecordBatch)*
  -> DoPut
  -> server decode
  -> PutResult*
```

## Where To Read the Code

For query/fetch:

- `arrow-flight/src/client.rs`
- `arrow-flight/src/decode.rs`
- `arrow-flight/tests/client.rs`

For ingest:

- `arrow-flight/src/encode.rs`
- `arrow-flight/src/streams.rs`
- `arrow-flight/src/sql/client.rs`
- `arrow-flight/tests/encode_decode.rs`

For SQL-flavored versions:

- `arrow-flight/src/sql/mod.rs`
- `arrow-flight/src/sql/client.rs`
- `arrow-flight/src/sql/server.rs`
- `arrow-flight/examples/flight_sql_server.rs`

## Final Mental Model

If you remember only one sentence from the whole tutorial, use this one:

> Flight is a control protocol for discovering, streaming, uploading, and managing Arrow IPC datasets over gRPC.

Everything in the crate follows from that sentence:

- protobuf types define the conversation,
- Arrow IPC defines the payload,
- tonic streams define the transport mechanics,
- helper layers make the protocol usable without hiding its shape.
