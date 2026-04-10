# 4. Schema, Fields, And Logical Types

This chapter answers the logical question:

**Once bytes are arranged correctly in memory, what meaning do those bytes have?**

Arrow answers that with `DataType`, `Field`, and `Schema`.

## Why schema is separated from physical layout

One of the most important architectural choices in Arrow is the separation between:

- logical meaning,
- physical realization.

That split makes several things possible:

- schema negotiation without data ownership,
- IPC/FFI schema exchange independent of buffers,
- planning and type reasoning before any bytes are loaded,
- multiple physical representations for related logical concepts.

If schema and buffer layout were fused into one giant type, all of those become harder.

## `DataType` is the logical universe

The `DataType` enum describes the meaning of values:

- primitive scalars,
- temporal values,
- decimals,
- strings and binaries,
- nested structures,
- dictionaries,
- unions,
- run-end encoded arrays,
- view-based variable-width types.

The important point is that `DataType` is not just an enum for display. Its docs capture semantic policies and edge-case meaning that later layers rely on.

## Why timestamp semantics are documented so carefully

The timestamp documentation is one of the best places to see Arrow's philosophy.

It distinguishes:

- timestamps with a timezone: physical instants relative to the Unix epoch,
- timestamps without a timezone: wall-clock values in an unspecified zone.

That is a policy choice in favor of semantic precision. The type system is trying to prevent the common but incorrect collapse of:

```text
"time with explicit timezone meaning"
and
"time value with no timezone meaning"
```

into one ambiguous representation.

This is why metadata-only timezone conversion is allowed in one case and not the other.

## Why `Date64` is handled pragmatically

The docs explain that `Date64` historically stores date-like values in milliseconds since epoch, and exact-day semantics are desirable, but the crate does not strictly validate divisibility by `86_400_000`.

That reveals another important policy:

- document semantic expectations clearly,
- preserve compatibility,
- avoid expensive or surprising validation in hot paths when the value is limited.

This is a recurring Arrow Rust theme: correctness through invariant boundaries and documentation, not gratuitous runtime cost everywhere.

## Why `Field` is more than name + type

A `Field` carries:

- name,
- `DataType`,
- nullability,
- metadata.

This is the smallest unit of logical contract that still matters operationally.

Why nullability is here:

- logical nullability is part of meaning, not just a physical storage detail.

Why metadata is here:

- systems often need semantic hints without inventing new physical types.

For nested data, fields recursively define the logical tree that physical arrays will realize.

## Why schema equality is opinionated

One subtle implementation detail is especially revealing: logical field/schema equality intentionally ignores deprecated IPC-era dictionary id baggage.

That tells you the maintainers are protecting a distinction between:

- logical equality users care about,
- transport-specific bookkeeping users should not have to treat as schema meaning.

This is a good example of an implementation refusing to let historical serialization details leak into the logical model.

## Why `Schema` does not know row count or buffers

A `Schema` tells you:

- what columns exist,
- what each column means,
- how nested children are structured,
- what metadata is attached.

It does **not** tell you:

- how many rows there are,
- where buffers live,
- whether there is IPC framing,
- whether the data came from FFI, Parquet, or in-memory builders.

That is intentional. A schema is a reusable logical contract across many batches and transport boundaries.

## Logical tree versus physical nodes

Example:

```text
Schema
└── events: List<Struct<
      user_id: Int64,
      payload: Utf8
    >>

Physical realization
  List node
    offsets buffer
    child Struct node
      child Int64 node
      child Utf8 node
```

The schema gives meaning to the tree. `ArrayData` and typed arrays tell you how each node is physically realized.

## Why the schema layer has its own crate

Because many important tasks are “schema-only” tasks:

- planning,
- negotiation,
- metadata transformation,
- FFI schema exchange,
- IPC schema exchange,
- logical equality checks.

Putting schema in its own crate keeps those tasks decoupled from allocation and compute.

## Reimplementation checklist

To build another Arrow implementation, you need:

1. a logical type universe,
2. fields carrying nullability and metadata,
3. schemas as reusable dataset-level contracts,
4. a clean separation from physical buffers.

That is the meaning layer of the system.

## Source markers

- `DataType`:
  [`arrow-schema/src/datatype.rs`](../../arrow-schema/src/datatype.rs#L96)
- `Field`:
  [`arrow-schema/src/field.rs`](../../arrow-schema/src/field.rs#L49)
- `SchemaBuilder` and `Schema`:
  [`arrow-schema/src/schema.rs`](../../arrow-schema/src/schema.rs#L29),
  [`arrow-schema/src/schema.rs`](../../arrow-schema/src/schema.rs#L187)
