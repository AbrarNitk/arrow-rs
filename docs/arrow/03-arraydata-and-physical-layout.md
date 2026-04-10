# 3. `ArrayData`: The Physical Invariant Boundary

This chapter answers the next major question:

**When do a bunch of buffers stop being “some bytes” and become “a valid Arrow array”?**

`ArrayData` is the boundary where that transition happens.

## Why `ArrayData` exists

If Arrow only exposed typed arrays such as `Int32Array` or `StringArray`, many important tasks would become awkward:

- generic validation,
- IPC import/export,
- FFI import/export,
- dynamic readers,
- nested layout reasoning before downcasting,
- building arrays from raw buffers.

So the project needs one universal physical representation that can describe:

- fixed-width arrays,
- variable-width arrays,
- nested arrays,
- dictionary arrays,
- unions,
- view arrays,
- nullability and slicing state.

That representation is `ArrayData`.

## What `ArrayData` stores

At a high level:

- `data_type`
- `len`
- `offset`
- `buffers`
- `child_data`
- `nulls`

That sounds simple, but it is carrying almost the entire physical contract of Arrow.

## The key design split

`ArrayData` says:

- what physical buffers exist,
- how many logical elements are visible,
- where slicing starts,
- which child arrays belong to this node,
- whether nullability exists.

It does **not** decide:

- what high-level methods the user sees,
- how kernels downcast typed arrays,
- how IPC frames messages,
- how schema should be displayed.

Those are higher layers.

## Core picture: variable-width array

```text
ArrayData
  data_type = Utf8
  len       = 3
  offset    = 0
  nulls     = validity bitmap
  buffers   = [offsets, values]
  children  = []

offsets = [0, 5, 5, 9]
values  = "hellorust"
nulls   = [1, 0, 1]
```

This one object is enough for generic code to understand the physical shape of the array without knowing about `StringArray` methods.

## Core picture: nested array

For `List<Int32>`:

```text
Parent ArrayData (List)
  buffers   = [offsets]
  child_data= [child Int32 ArrayData]

offsets = [0, 2, 2, 5]

Child Int32 ArrayData
  values = [10, 11, 20, 21, 22]

logical rows:
  row0 -> child[0..2]
  row1 -> child[2..2]
  row2 -> child[2..5]
```

The important thing to notice is that the parent does not duplicate child values. Nested structure is represented by composition of array nodes, not by inventing a different buffer model.

## Why `new_buffers` is such an important map

The helper `new_buffers(data_type, capacity)` in `arrow-data/src/data.rs` is one of the fastest ways to understand Arrow's physical storage policy.

It tells you, per logical type:

- primitive types -> one values buffer,
- booleans -> one bitmap buffer,
- `Utf8` / `Binary` -> offsets + values,
- `Utf8View` / `BinaryView` -> one fixed-width views buffer,
- `List` -> offsets, child arrays,
- `Struct` -> children only,
- `DenseUnion` -> type ids + offsets + children.

That is the real “physical type table” of the implementation.

## Why `offset` is subtle

The `offset` field is easy to misunderstand.

It applies to:

- `buffers`
- `child_data`

but not to `nulls` in the same way. The null buffer already represents the visible logical length.

Why this matters:

- slicing must be cheap,
- slices should usually share underlying storage,
- but shared storage cannot mean the null semantics become ambiguous.

This is a good example of where the representation is optimized for zero-copy slicing without making null handling unsafe.

## Why `try_new` and `new_unchecked` both exist

The existence of both constructors tells you where the trust boundary is.

### `try_new`

Use when:

- buffers come from untrusted or external sources,
- a malformed layout is possible,
- you want Arrow to validate the physical structure.

### `unsafe new_unchecked`

Use when:

- the caller already knows the invariants hold,
- validation cost would be redundant,
- the data is internal or otherwise trusted.

This is one of the deepest architectural choices in Arrow Rust:

- high-level safety by default,
- explicit unsafe fast paths for trusted construction,
- centralized validation instead of ad hoc checks everywhere.

## Why validation is centralized

`validate_data()` checks questions like:

- are there enough buffers?
- are buffer lengths large enough for `len + offset`?
- do offsets fit the child/value buffers?
- do child arrays agree with the parent layout?
- is null buffer length sufficient?

This is not just defensive programming. It allows the rest of the system to rely on strong assumptions once an `ArrayData` exists.

That is why `Array` being `unsafe` later on makes sense: many typed-array operations assume `ArrayData`-level invariants already hold.

## Why `layout(data_type)` matters so much

`layout(data_type)` returns a `DataTypeLayout` with `BufferSpec`s.

This function is the machine-readable form of Arrow's physical policy:

- which buffers exist,
- fixed-width versus variable-width,
- required alignment,
- whether nulls are allowed,
- whether buffer count is variadic.

If you wanted to generate validators, importers, or generic builders automatically, this is one of the most important functions in the codebase.

## Why `ArrayData` is the right abstraction boundary

It sits at exactly the right layer:

- low enough to describe all array layouts,
- high enough to hide raw allocator details,
- generic enough for IPC/FFI,
- structured enough for validation.

This is why many later abstractions reduce to `ArrayData` or are rebuilt from it.

## Reimplementation checklist

To build another Arrow implementation, you would need:

1. a universal physical array descriptor,
2. a table mapping logical types to buffer layouts,
3. centralized validation,
4. explicit slicing semantics,
5. a trust boundary between checked and unchecked construction.

That is the real job of `ArrayData`.

## Source markers

- `ArrayData` definition:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L205)
- Checked and unchecked construction:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L273),
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L315)
- Canonical buffer creation:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L59)
- `layout(data_type)` and `DataTypeLayout`:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1690)
- `BufferSpec`:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1886)
- Builder:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1984)
