# 02. Schema And Nesting

This is the chapter where Parquet stops looking like “column chunks on disk” and starts looking like “a system for shredding and reconstructing nested rows.”

The key question is:

**How can a nested logical record be stored in leaf columns without losing enough information to reconstruct the original structure later?**

Parquet's answer is:

- flatten the schema tree into primitive leaves,
- store one physical column per primitive leaf,
- attach repetition and definition level streams so the reader can reconstruct nulls and nesting boundaries.

## Start from the problem

Suppose your logical rows look like this:

```text
row 0: { id: 1, user: { name: "a", phones: ["111", "222"] } }
row 1: { id: 2, user: null }
row 2: { id: 3, user: { name: null, phones: [] } }
```

If you store only the leaf values:

```text
id column          = [1, 2, 3]
user.name column   = ["a", ?]
phones.number      = ["111", "222"]
```

you have already lost critical information:

- Was `user` null, or was `user` present but `name` null?
- Did `"111"` and `"222"` belong to the same row or to two different rows?
- Was `phones` empty or missing?

That missing information is exactly why Parquet has levels.

## The schema tree

In `arrow-rs`, schema is represented by `schema::types::Type`:

- `PrimitiveType` for leaf fields,
- `GroupType` for nested structure,
- the root message is a `GroupType` without repetition.

That tree is the logical contract. It is not yet the physical storage plan.

## From logical tree to physical leaf columns

Example schema:

```text
message root
├─ required id: INT64
└─ optional user
   ├─ optional name: BYTE_ARRAY (STRING)
   └─ repeated phones
      └─ optional number: BYTE_ARRAY (STRING)
```

Logical tree view:

```text
root
├─ id
└─ user
   ├─ name
   └─ phones
      └─ number
```

Physical leaf-column view:

```text
0 -> id
1 -> user.name
2 -> user.phones.number
```

This is the first huge conceptual jump in Parquet:

- the user sees nested fields,
- the file stores primitive leaves,
- the meaning of those leaves is recovered through path + levels.

## `SchemaDescriptor` is the shredding boundary

`SchemaDescriptor` is where the schema becomes operational for IO.

It performs a depth-first walk and produces one `ColumnDescriptor` per primitive leaf. For each leaf it records:

- the full column path,
- the primitive physical type,
- `max_def_level`,
- `max_rep_level`,
- mapping information back to higher-level fields.

This is not just convenience metadata. It is the exact information the page layer needs to know how many level bits are required and how to interpret them.

## Definition levels answer “how far did we get?”

A definition level tells the reader how much of the path to the leaf is actually present.

For `user.phones.number`, a record may fail to reach the leaf for several different reasons:

- `user` may be null,
- `user` may exist but `phones` may be empty,
- `phones` may have an element whose `number` is null,
- the leaf value may be present.

Definition level distinguishes those states.

### Example intuition

Imagine the path:

```text
root
└─ optional user
   └─ repeated phones
      └─ optional number
```

One possible interpretation is:

- def = 0: path breaks before `user`
- def = 1: `user` exists, but no concrete `number` value is defined here
- def = 2: repeated position exists, but `number` itself is null
- def = 3: full leaf value exists

The exact integers depend on the schema, but the principle never changes:

**definition level counts how much of the path is defined.**

## Repetition levels answer “did we start a new record?”

Once a field is repeated, values alone are not enough to know row boundaries.

Repetition level tells the reader whether the current leaf entry:

- starts a new logical record,
- or continues the same repeated structure as the previous entry.

This is what lets a flat leaf stream represent lists and repeated groups.

## Worked example: why both level streams are needed

Using the earlier rows:

```text
row 0: user.phones = ["111", "222"]
row 1: user = null
row 2: user.phones = []
```

One possible leaf stream for `user.phones.number` looks like:

```text
entry    value   def   rep   interpretation
-----    -----   ---   ---   -----------------------------------------
0        111      3     0    start row 0, first phone present
1        222      3     1    continue row 0, another phone present
2        -        0     0    start row 1, user missing entirely
3        -        1     0    start row 2, user present, phones empty
```

Read that table carefully:

- `rep` tells you whether `"222"` belongs to the same row as `"111"`,
- `def` tells you why entries 2 and 3 have no value,
- together they let the reader rebuild the original rows.

If you removed either stream:

- without `rep`, repeated boundaries are ambiguous,
- without `def`, null versus empty versus absent is ambiguous.

That is the core nested-data insight in Parquet.

## Why levels are encoded per leaf column

Because reconstruction happens from leaf streams upward.

Each primitive leaf column must carry enough information for a reader to say:

- which logical records these entries belong to,
- whether each leaf value exists,
- and how far up the nested path to materialize null or empty structure.

This is why `max_def_level` and `max_rep_level` are properties of the leaf `ColumnDescriptor`, not just of a high-level group.

## Why `max_def_level == 0` or `max_rep_level == 0` matters

These are not just small optimizations.

- `max_def_level == 0` means the field is required all the way down, so no definition level stream is needed.
- `max_rep_level == 0` means the path contains no repetition, so no repetition level stream is needed.

In other words, the schema tells the page layer when certain byte streams can be omitted entirely.

## Reimplementation checklist

To reimplement schema shredding correctly, you need:

1. a schema tree with repetition/optionality on each node,
2. a DFS walk that enumerates primitive leaves,
3. a path representation per leaf,
4. a rule to compute `max_def_level` and `max_rep_level`,
5. encoding and decoding logic that treats those levels as first-class streams.

Without those pieces, nested Parquet is impossible to reconstruct correctly.

## Source markers

- Logical schema representation:
  [`parquet/src/schema/types.rs`](../../parquet/src/schema/types.rs#L29)
- `ColumnDescriptor` carrying level information:
  [`parquet/src/schema/types.rs`](../../parquet/src/schema/types.rs#L841)
- `SchemaDescriptor` flattening the tree into leaf columns:
  [`parquet/src/schema/types.rs`](../../parquet/src/schema/types.rs#L1008)
- Metadata carrying schema descriptor:
  [`parquet/src/file/metadata/mod.rs`](../../parquet/src/file/metadata/mod.rs#L486)
- Level encoder:
  [`parquet/src/encodings/levels.rs`](../../parquet/src/encodings/levels.rs#L17)
- Low-level record reader wiring levels to decode:
  [`parquet/src/column/reader.rs`](../../parquet/src/column/reader.rs),
  [`parquet/src/record/triplet.rs`](../../parquet/src/record/triplet.rs)
