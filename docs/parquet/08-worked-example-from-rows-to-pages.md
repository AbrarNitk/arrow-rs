# 08. Worked Example: From Logical Rows To Page Bytes And Back

This chapter is the synthesis chapter.

Its purpose is to answer the hardest practical question in the whole tutorial:

**If I start with a small nested dataset, what exactly happens as Parquet shreds it into leaf columns, writes pages, records footer metadata, and then reconstructs it again on read?**

If you can explain this chapter fluently, you understand Parquet much more like an implementer than like a casual user.

## The example schema

We will use a deliberately small nested schema with one required scalar, one optional scalar, and one repeated nested value:

```text
message root
├─ required id: INT64
└─ optional user
   ├─ optional name: BYTE_ARRAY (STRING)
   └─ repeated phones
      └─ optional number: BYTE_ARRAY (STRING)
```

This is not an arbitrary example. It was chosen because it exercises all the key cases:

- required leaf: `id`
- optional leaf behind an optional parent: `user.name`
- repeated + optional leaf: `user.phones.number`

That means we will see all of:

- no level streams,
- definition levels only,
- definition + repetition levels together.

## The example logical rows

We will write three rows:

```text
row 0: { id: 1, user: { name: "alice", phones: ["111", "222"] } }
row 1: { id: 2, user: null }
row 2: { id: 3, user: { name: null, phones: [] } }
```

Keep these rows in mind. Every byte we discuss is just a compact way of preserving the information in those three logical records.

## Step 1: shred the schema into leaf columns

The logical tree becomes three primitive leaf columns:

```text
leaf 0 -> id
leaf 1 -> user.name
leaf 2 -> user.phones.number
```

`SchemaDescriptor` and `ColumnDescriptor` are the implementation boundary where this flattening becomes concrete.

Why the flattening is necessary:

- Parquet stores values by primitive leaf for scan efficiency,
- physical encoders and compressors operate on primitive streams,
- page and chunk metadata are defined per leaf column.

Without flattening, Parquet would lose its core “columnar leaf stream” property.

## Step 2: compute max definition and repetition levels

Now ask: for each leaf, what structural information must be recorded per physical entry?

### Leaf `id`

Path:

```text
root -> id
```

This leaf is required and not repeated.

So:

- `max_def_level = 0`
- `max_rep_level = 0`

Meaning:

- no definition-level stream is needed,
- no repetition-level stream is needed,
- every physical entry is just a value.

### Leaf `user.name`

Path:

```text
root -> optional user -> optional name
```

This leaf can fail to exist in two different places:

- `user` is null,
- `user` exists but `name` is null.

So this leaf needs definition levels, but not repetition levels:

- `max_def_level = 2`
- `max_rep_level = 0`

One useful interpretation is:

- `def = 0`: `user` missing
- `def = 1`: `user` exists, `name` missing
- `def = 2`: `name` present

### Leaf `user.phones.number`

Path:

```text
root -> optional user -> repeated phones -> optional number
```

This leaf must distinguish:

- `user` missing,
- `user` present but `phones` empty,
- a repeated phone slot whose `number` is null,
- a repeated phone slot with a real value,
- whether successive values belong to the same row or to different rows.

So this leaf needs both level streams:

- `max_def_level = 3`
- `max_rep_level = 1`

One useful interpretation is:

- `def = 0`: `user` missing
- `def = 1`: `user` present, but no phone value materialized here
- `def = 2`: phone slot exists, but `number` is null
- `def = 3`: `number` is present

And:

- `rep = 0`: start a new row
- `rep = 1`: continue the repeated `phones` field in the current row

That single pair of integers is the core of nested Parquet reconstruction.

## Step 3: produce the physical leaf streams

### Leaf `id`

This one is trivial:

```text
values = [1, 2, 3]
```

There are no level streams because the schema made them unnecessary.

### Leaf `user.name`

Per row:

- row 0 -> `"alice"` present
- row 1 -> `user` is null
- row 2 -> `user` present, `name` is null

So the physical stream is:

```text
def levels = [2, 0, 1]
values     = ["alice"]
```

Notice the asymmetry:

- there are 3 structural positions,
- but only 1 actual value.

This is one of the most important habits to develop when reading Parquet code:

**level count and value count are not the same thing.**

### Leaf `user.phones.number`

Per row:

- row 0 -> two values: `"111"`, `"222"`
- row 1 -> `user` missing
- row 2 -> `phones` empty

So the physical stream is:

```text
entry    value   def   rep   meaning
-----    -----   ---   ---   -----------------------------------------
0        111      3     0    row 0 starts, first phone value present
1        222      3     1    same row 0, another phone value present
2        -        0     0    row 1 starts, user missing
3        -        1     0    row 2 starts, user present, phones empty
```

So the leaf streams are:

```text
rep levels = [0, 1, 0, 0]
def levels = [3, 3, 0, 1]
values     = ["111", "222"]
```

Again, the asymmetry is the whole point:

- 4 structural entries,
- only 2 actual values.

## Why this is enough to reconstruct the original rows

Read the `phones.number` stream from left to right:

1. `rep=0, def=3, value=111`
   new row, user exists, first phone is `"111"`

2. `rep=1, def=3, value=222`
   same row, second phone is `"222"`

3. `rep=0, def=0`
   new row, user missing entirely

4. `rep=0, def=1`
   new row, user exists, but there is no phone value here because the list is empty

That is exactly the original logical information.

This is why levels are not an awkward side mechanism. They are the mechanism.

## Step 4: what the writer computes in `arrow-rs`

On the Arrow-writing path, `parquet/src/arrow/arrow_writer/levels.rs` computes this structural information by walking Arrow arrays and their null/offset structure.

The important pieces are:

- `calculate_array_levels`
- `LevelInfoBuilder::write_struct`
- `LevelInfoBuilder::write_list`
- `LevelInfoBuilder::write_leaf`
- `ArrayLevels`

`ArrayLevels` holds exactly the information the low-level column writer needs:

- `def_levels`
- `rep_levels`
- `non_null_indices`
- `max_def_level`
- `max_rep_level`
- leaf array reference

That `non_null_indices` vector is an especially important behind-the-scenes detail. It is how the writer keeps the compact leaf value array aligned with the longer structural level stream.

## Why `non_null_indices` exists

Look again at `user.name`:

```text
def levels = [2, 0, 1]
values     = ["alice"]
```

The leaf values are not stored in a vector with placeholders for nulls. The writer wants the compact list of actual values:

```text
actual non-null values = ["alice"]
```

But the page writer still needs to know which structural positions consume values.

That is what `non_null_indices` solves:

- level streams keep the full structural timeline,
- `non_null_indices` maps compact non-null values back to the positions where `def == max_def_level`.

This is why Parquet writers do not need to materialize fake null payloads just to preserve alignment.

## Step 5: hand the streams to the low-level column writer

At the low-level writer layer, `ColumnWriterImpl::write_batch_internal` receives:

- compact values,
- optional definition levels,
- optional repetition levels.

Then it does three crucial things:

1. validates level presence when the schema requires them,
2. counts how many actual values should be written by checking `def == max_def_level`,
3. encodes rep/def streams separately from the value stream.

This is why the writer API accepts both values and levels explicitly: the writer cannot infer levels from the compact values alone.

## Step 6: build page bodies

For a simple one-page chunk, the body shapes are conceptually:

### `id`

```text
page body = [encoded values]
```

### `user.name`

```text
page body = [def levels][encoded values]
```

### `user.phones.number`

```text
page body = [rep levels][def levels][encoded values]
```

For Data Page V2, the header also records:

- `repetition_levels_byte_length`
- `definition_levels_byte_length`
- `num_values`
- `num_rows`
- `num_nulls`

This is exactly the information a reader needs to split the body back into its three logical streams.

## Why `num_values`, `num_rows`, and `num_nulls` are all needed

These counters are not redundant.

- `num_values` counts structural entries in the page.
- `num_rows` counts logical rows represented.
- `num_nulls` counts null leaf outcomes.

For flat required data, these may collapse to the same number.
For nested nullable data, they diverge.

That divergence is why nested Parquet is fundamentally not “just values plus compression.”

## Step 7: what the footer records for this chunk

After the page is written, the writer can finally record chunk metadata such as:

- chunk offset,
- compressed size,
- data page offset,
- optional dictionary page offset,
- number of values,
- codec,
- encodings,
- statistics,
- optional page index offsets.

For our `user.phones.number` chunk, the footer does **not** need to store the full level stream again. It only needs to store enough metadata for the reader to find the pages and know how to decode them.

That is another important design idea:

- pages carry the detailed structural timeline,
- footer metadata carries the navigation and planning information.

## Step 8: what the reader does with the bytes

On read, the path is the reverse:

1. footer finds the chunk,
2. page reader finds the page,
3. page header tells how to split the body,
4. level decoders reconstruct rep/def arrays,
5. value decoder reconstructs the compact value buffer,
6. record reconstruction combines them.

## `ColumnReaderImpl::read_records` and why it returns three counters

`read_records` returns:

- records read,
- non-null values read,
- levels read.

Those are separate because nested nullable data makes them separate.

For our `user.phones.number` example:

```text
rep levels = [0, 1, 0, 0]
def levels = [3, 3, 0, 1]
values     = ["111", "222"]
```

the reader may observe:

- 3 logical records,
- 2 non-null values,
- 4 levels.

That is not an edge case. That is normal nested Parquet behavior.

## `TripletIter`: the reconstruction lens

The record API's `TripletIter` is a very good way to understand the reconstruction semantics because it makes the current triplet explicit:

- current definition level,
- current repetition level,
- current value if non-null.

It also makes one crucial rule obvious:

```text
current_def_level < max_def_level  => current logical value is null / not fully defined
```

That single comparison is how the reader decides whether there is a real leaf value at the current structural position.

## Why value spacing is needed on read

In `TypedTripletIter::read_next`, if the number of values read is smaller than the number of levels read, the implementation spaces the compact values back into positions aligned with the level stream.

That is the read-side mirror of `non_null_indices` on the write side.

Write side:

- compact values
- plus metadata about where real values occur

Read side:

- compact decoded values
- plus level stream
- then re-space values into triplet positions where `def == max_def_level`

This symmetry is one of the most elegant parts of the implementation.

## End-to-end reconstruction table

For the `user.phones.number` leaf:

```text
triplet   rep   def   value   reconstructed effect
-------   ---   ---   -----   -----------------------------------------
0          0     3    111     start row 0, append phone "111"
1          1     3    222     continue row 0, append phone "222"
2          0     0    -       start row 1, materialize user = null
3          0     1    -       start row 2, materialize user present, phones = []
```

This is the original logical dataset recovered from one leaf stream plus its sibling leaves.

## How this example connects to the codebase

This chapter sits across four different implementation layers:

```text
Arrow arrays / RecordBatch
   │
   ▼
arrow_writer/levels.rs
  computes def/rep + non_null_indices
   │
   ▼
column/writer/mod.rs
  turns them into page bodies
   │
   ▼
file metadata
  records where those pages live
   │
   ▼
column/reader.rs + record/triplet.rs
  decode and reconstruct
```

That is the real systems story of Parquet in `arrow-rs`.

## What policies now make sense after this example

After walking one nested example end to end, the main design choices should feel justified:

- Footer at end:
  offsets and sizes are only known after pages are written.

- Leaf-column storage:
  primitive streams are the right unit for compression and column pruning.

- Row groups:
  preserve row alignment across columns while keeping partitions large enough for throughput.

- Per-page headers:
  each page must be independently decodable.

- Definition and repetition levels:
  nested reconstruction is impossible without them.

- Optional page indexes:
  fine-grained pruning is useful, but not worth forcing into every file.

- Separate value count and level count:
  nullable nested data naturally makes them diverge.

## Reimplementation checklist

If you had to rewrite Parquet support from scratch, this chapter says you need to implement these exact moving parts:

1. schema flattening into primitive leaves,
2. max def/rep level computation,
3. shredding logical rows into level streams + compact values,
4. page-body assembly with explicit level/value boundaries,
5. footer metadata that points back to chunk and page structures,
6. read-side level decoding,
7. read-side value spacing and triplet reconstruction.

Anything less is not a full Parquet implementation.

## Source markers

- Arrow-side level computation entry point:
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L55)
- List, struct, and leaf level-writing logic:
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L306),
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L476),
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L625)
- `ArrayLevels` and `non_null_indices`:
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L731)
- Concrete expected level vectors in tests:
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L1023),
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L1170),
  [`parquet/src/arrow/arrow_writer/levels.rs`](../../parquet/src/arrow/arrow_writer/levels.rs#L1648)
- Column writer consuming values + levels:
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs#L667),
  [`parquet/src/column/writer/mod.rs`](../../parquet/src/column/writer/mod.rs#L1102)
- Column reader reconstructing records / values / levels:
  [`parquet/src/column/reader.rs`](../../parquet/src/column/reader.rs#L139),
  [`parquet/src/column/reader.rs`](../../parquet/src/column/reader.rs#L192)
- Record API triplet reconstruction:
  [`parquet/src/record/triplet.rs`](../../parquet/src/record/triplet.rs#L41),
  [`parquet/src/record/triplet.rs`](../../parquet/src/record/triplet.rs#L168)
