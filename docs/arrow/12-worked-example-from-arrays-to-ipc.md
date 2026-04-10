# 12. Worked Example: From Logical Values To Buffers, Arrays, RecordBatch, And IPC

This is the synthesis chapter.

Its purpose is to answer the hardest practical question in the tutorial:

**If I start with a small nested dataset, what exactly happens as Arrow turns it into buffers, array nodes, a `RecordBatch`, and then an IPC artifact that can be reconstructed again?**

If you can explain this chapter cleanly, you understand Arrow much more like an implementer than like a library user.

## The example schema

We will use three columns:

```text
id:    Int32                     required
name:  Utf8                      nullable
tags:  List<Utf8>                nullable list, nullable elements not used here
```

Why this example:

- `id` shows the simplest fixed-width case,
- `name` shows classic variable-width + nullability,
- `tags` shows nesting via offsets + child arrays.

That is enough to exercise most of Arrow's core mechanics without becoming unreadable.

## The example rows

```text
row 0: { id: 1, name: "alice", tags: ["x", "y"] }
row 1: { id: 2, name: null,    tags: null }
row 2: { id: 3, name: "",      tags: [] }
```

Keep these rows in mind. Everything below is just a compact physical encoding of these logical values.

## Step 1: `id` becomes the ideal Arrow array

Logical values:

```text
[1, 2, 3]
```

Physical shape:

```text
data_type = Int32
len       = 3
nulls     = None
buffers   = [values]

values buffer = [1, 2, 3]
```

There is no null bitmap because the field is required. This is the cleanest Arrow case and the one many kernels are optimized around.

## Step 2: `name` becomes offsets + values + null bitmap

Logical values:

```text
["alice", null, ""]
```

Physical shape:

```text
data_type = Utf8
len       = 3
nulls     = [1, 0, 1]
buffers   = [offsets, values]

offsets = [0, 5, 5, 5]
values  = [a l i c e]
```

Interpretation:

- slot 0 -> values[0..5] = "alice"
- slot 1 -> null because validity bit is 0
- slot 2 -> values[5..5] = ""

Notice what Arrow is doing here:

- the null does not remove an offset entry,
- the empty string naturally appears as equal adjacent offsets,
- the values buffer remains compact and contiguous.

That is why offsets + bitmap is such a powerful design.

## Step 3: `tags` becomes parent offsets plus child array

Logical values:

```text
row 0 -> ["x", "y"]
row 1 -> null
row 2 -> []
```

Parent list node:

```text
data_type = List<Utf8>
len       = 3
nulls     = [1, 0, 1]
buffers   = [offsets]

list offsets = [0, 2, 2, 2]
```

Child values node is itself a `Utf8` array:

```text
child data_type = Utf8
child len       = 2
child nulls     = None
child buffers   = [offsets, values]

child offsets = [0, 1, 2]
child values  = [x y]
```

Interpretation:

- row 0 -> child[0..2] = ["x", "y"]
- row 1 -> null because list validity bit is 0
- row 2 -> child[2..2] = []

This is the central nested Arrow trick:

- parent arrays describe structure,
- child arrays carry actual values,
- composition replaces special-case nested object layouts.

## Step 4: assemble the generic `ArrayData` view

At the `ArrayData` level, the three columns look like:

```text
id:
  buffers   = [values]
  child_data= []

name:
  buffers   = [offsets, values]
  child_data= []
  nulls     = bitmap

tags:
  buffers   = [list offsets]
  child_data= [child Utf8 ArrayData]
  nulls     = bitmap
```

This is why `ArrayData` is the physical invariant boundary: every typed array above it reduces to this shape.

## Step 5: lift to typed arrays

Now those generic layouts become:

- `Int32Array`
- `StringArray`
- `ListArray<StringArray>`

The typed wrappers give ergonomic access, but they do not change the underlying physical story.

That is the right relationship to remember:

```text
typed arrays
  = interpreted views over validated physical layout
```

not:

```text
typed arrays
  = a completely different storage model
```

## Step 6: align the columns into a `RecordBatch`

Now we can create a batch:

```text
schema:
  id:   Int32 not null
  name: Utf8 nullable
  tags: List<Utf8> nullable

columns:
  id.len   = 3
  name.len = 3
  tags.len = 3
```

`RecordBatch` is where Arrow says:

- these arrays share one schema,
- their lengths match,
- therefore position `i` across all columns defines row `i`.

The rows are not stored physically anywhere. They are reconstructed by aligned position across columns.

## Step 7: what IPC must preserve

When writing this batch to Arrow IPC, the writer must preserve enough information to reconstruct:

- the schema tree,
- the buffer ordering,
- the child relationships,
- null buffers,
- offsets and values buffers,
- alignment and optional compression.

It does this by splitting the message into:

- FlatBuffer metadata,
- Arrow body buffers.

Conceptually:

```text
RecordBatch
   │
   ├─ schema metadata (FlatBuffer)
   ├─ field nodes
   ├─ buffer metadata
   └─ aligned body buffers
```

The critical point is that the body buffers remain Arrow-shaped. IPC does not invent a second logical array encoding for the payload.

## Step 8: why the reader needs schema + nodes + buffers

On read, `RecordBatchDecoder::create_array` reconstructs arrays recursively.

For our example it must know:

- `id` is primitive fixed-width,
- `name` needs three buffers in IPC terms: nulls, offsets, values,
- `tags` needs a parent list node plus child array reconstruction,
- child `Utf8` itself again needs nulls/offsets/values.

That is why Arrow separates:

- schema,
- field nodes,
- buffers.

No one of these alone is sufficient to reconstruct the batch.

## Step 9: where zero-copy becomes real

If the IPC file is memory-mapped and buffer alignment is acceptable, reconstructed arrays can point directly into the mapped region.

This is the practical meaning of Arrow's design:

- typed arrays can be rebuilt mostly from metadata and slices into existing bytes,
- not from per-value deserialization.

The zero-copy IPC example in the repo shows exactly this path.

## Why this example justifies the Arrow policies

After walking one dataset through the system, the main design choices should feel justified:

- Buffers instead of objects:
  all three columns are represented as a few contiguous buffers rather than many little heap allocations.

- Bitmap nullability:
  `name` and `tags` preserve dense payload buffers while still representing nulls.

- Offsets for variable-width data:
  both empty string and empty list are natural, cheap cases.

- Child arrays for nesting:
  `tags` does not require inventing a second nested storage model.

- `ArrayData` as the invariant boundary:
  all later layers reduce to it.

- `RecordBatch` as the tabular unit:
  rows are aligned by index across columns without a row layout.

- IPC preserving Arrow layout:
  transport does not throw away the memory model.

## Reimplementation checklist

If you wanted to build another Arrow implementation from scratch, this chapter says you need to implement:

1. fixed-width arrays,
2. variable-width arrays with offsets,
3. nested arrays with child nodes,
4. null bitmaps,
5. generic physical layout descriptors,
6. typed array views,
7. batch alignment,
8. IPC framing that preserves the underlying buffer order.

That is the end-to-end system.

## Source markers

- `ArrayData` and constructors:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L205),
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L315)
- `Array` safety boundary:
  [`arrow-array/src/array/mod.rs`](../../arrow-array/src/array/mod.rs#L82)
- Variable-width arrays:
  [`arrow-array/src/array/byte_array.rs`](../../arrow-array/src/array/byte_array.rs#L87)
- `RecordBatch`:
  [`arrow-array/src/record_batch.rs`](../../arrow-array/src/record_batch.rs#L224)
- IPC writer entry points:
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L51),
  [`arrow-ipc/src/writer.rs`](../../arrow-ipc/src/writer.rs#L200)
- IPC reader reconstruction:
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L78),
  [`arrow-ipc/src/reader.rs`](../../arrow-ipc/src/reader.rs#L977)
- Zero-copy example:
  [`arrow/examples/zero_copy_ipc.rs`](../../arrow/examples/zero_copy_ipc.rs#L1)
