# 4. Schema, Fields, And Logical Types

After physical layout comes the logical layer. Arrow separates:

- logical meaning: `DataType`, `Field`, `Schema`
- physical realization: buffers, offsets, nulls, children

This separation is deliberate. The same logical type may be represented in different ways during transport or conversion, but the schema layer is the common contract applications reason about.

## `DataType`: the logical universe

`arrow-schema/src/datatype.rs` defines the `DataType` enum. It includes:

- primitive fixed-width types,
- temporal types,
- decimal types,
- binary and string types,
- nested types like `List`, `Struct`, `Map`, `Union`,
- specialized types like dictionaries and run-end encoding,
- newer view-based variable-width types.

This enum is not just a catalog. Its documentation captures policy decisions and semantic constraints.

## Time semantics are treated carefully

The timestamp docs are especially important. They explain that:

- timestamps with a timezone represent physical instants relative to the Unix epoch,
- timestamps without a timezone are wall-clock values in an unknown timezone,
- changing between non-empty timezones is metadata-only,
- changing from no timezone to a timezone is not metadata-only.

That is a design choice to preserve semantic precision. The type system encodes "what kind of time meaning do you have?" instead of forcing all timestamps into one interpretation.

## Why `Date64` is documented so cautiously

The `Date64` docs explain that the format historically stores date-like values as milliseconds since epoch, but valid values should still represent exact days. The crate does not strictly enforce divisibility by `86_400_000` for performance and usability reasons, and recommends `Date32` or proper timestamp types when those fit better.

This is a good example of Arrow Rust's style:

- preserve compatibility with the spec and ecosystem,
- document edge semantics clearly,
- avoid costly validation in hot paths when the benefit is limited.

## `Field`: more than name + type

A `Field` contains:

- name,
- `DataType`,
- nullability,
- metadata.

For nested types, children are embedded inside the `DataType` and `Field` structure. A `List<Field>` or `Struct<Fields>` expresses the logical tree that the physical layout must later realize.

One subtle implementation detail matters: schema equality intentionally ignores deprecated IPC-specific dictionary id state. That choice keeps logical schema equality focused on application-visible meaning rather than transport-era baggage.

## `Schema`: dataset-level contract

A `Schema` is a collection of `Field`s plus metadata. It tells you:

- how many columns exist,
- what each column means,
- whether fields are nullable,
- what extra user or system metadata is attached.

It does **not** tell you:

- the number of rows,
- how buffers are allocated,
- whether the data is contiguous in one region,
- what IPC framing exists.

That separation is important. A schema is reusable across many batches.

## Logical tree versus physical leaves

For nested data, logical schema is tree-shaped while physical buffers are organized around concrete array nodes.

```text
Schema
└── events: List<Struct<
      user_id: Int64,
      payload: Utf8
    >>

Physical realization
  List array
    offsets buffer
    child Struct array
      child Int64 array
        values buffer
        null bitmap?
      child Utf8 array
        offsets buffer
        values buffer
        null bitmap?
```

The schema tells you what nodes exist; `ArrayData` and typed arrays tell you how those nodes are stored.

## Why the schema layer is isolated in its own crate

This separation makes several things easier:

- systems can negotiate schemas without owning data buffers,
- IPC and FFI can import/export schema independently of array bodies,
- logical planning can run without pulling in compute or buffer code,
- metadata transformations stay decoupled from allocation concerns.

## Source markers

- `DataType` and datatype docs:
  [`arrow-schema/src/datatype.rs`](../../arrow-schema/src/datatype.rs#L96)
- `Field` definition:
  [`arrow-schema/src/field.rs`](../../arrow-schema/src/field.rs#L49)
- `SchemaBuilder` and `Schema`:
  [`arrow-schema/src/schema.rs`](../../arrow-schema/src/schema.rs#L29),
  [`arrow-schema/src/schema.rs`](../../arrow-schema/src/schema.rs#L187)
