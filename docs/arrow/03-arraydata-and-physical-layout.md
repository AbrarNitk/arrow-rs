# 3. `ArrayData`: The Physical Invariant Boundary

Once you understand buffers, the next key type is `ArrayData`. This is where Arrow stops being "some bytes in memory" and becomes "a valid Arrow array layout."

`arrow-data` exists because the project needs one generic, type-independent representation that can describe every array shape.

## What `ArrayData` contains

At a high level, `ArrayData` contains:

- `data_type`
- `len`
- `offset`
- `buffers`
- `child_data`
- `nulls`

That is enough to represent primitive arrays, strings, lists, structs, unions, dictionary arrays, and view arrays.

## The core picture

```text
ArrayData
  data_type = Utf8
  len       = 3
  offset    = 0
  nulls     = validity bitmap
  buffers   = [offsets, values]
  children  = []

        ┌───────────────────────────┐
        │ offsets: [0, 5, 5, 9]     │
        └───────────────────────────┘
        ┌───────────────────────────┐
        │ values: "hellorust"       │
        └───────────────────────────┘
        ┌───────────────────────────┐
        │ nulls : 1 0 1             │
        └───────────────────────────┘
```

For a nested type like `List<Int32>`, the parent `ArrayData` points to child `ArrayData`:

```text
Parent ArrayData (List)
  buffers   = [offsets]
  child_data= [child Int32 ArrayData]

offsets = [0, 2, 2, 5]

Child Int32 ArrayData
  values = [10, 11, 20, 21, 22]

logical rows:
  row0 -> child[0..2] = [10, 11]
  row1 -> child[2..2] = []
  row2 -> child[2..5] = [20, 21, 22]
```

## Why `ArrayData` matters so much

Typed arrays in `arrow-array` are mostly wrappers around `ArrayData` or structured equivalents of the same parts.

That means `ArrayData` is the place where the library must answer hard questions:

- How many buffers does this `DataType` require?
- Which buffer positions mean what?
- Are offsets monotonic?
- Are buffer lengths large enough?
- Do child arrays agree with parent layout?
- Are null counts and null buffers coherent?

## `new_buffers` shows the canonical layout patterns

The helper `new_buffers(data_type, capacity)` in `arrow-data/src/data.rs` is a useful reading shortcut. It reveals the default physical buffer shape expected for each logical type:

- primitive types => one values buffer,
- booleans => one bit-packed buffer,
- `Utf8` / `Binary` => offsets + values,
- `Utf8View` / `BinaryView` => one `u128` views buffer,
- `List` => offsets + child array,
- `Struct` => children only,
- `Union::Dense` => type ids + offsets + children.

That function is effectively a compact map from logical type to memory shape.

## Offsets do not mean the same thing everywhere

One of the most important details in `ArrayData` is the `offset` field:

- it applies to `buffers`,
- it applies to `child_data`,
- it does **not** apply to `nulls` in the same way.

The code comments explicitly call this out. This matters because slicing an array should be cheap. A sliced array should often just adjust logical offset and length while still referencing the same underlying buffers.

## Validation versus unchecked construction

The API exposes both:

- `try_new(...)`
- `unsafe new_unchecked(...)`

This is the core trust boundary.

Use `try_new` when data may be malformed. It validates the physical layout. Use `new_unchecked` only when the caller can already prove the invariants.

That split is one of the deepest architectural decisions in Arrow Rust:

- safety at the public boundary,
- optional unchecked fast paths for internal or trusted code,
- centralized validation instead of scattered ad hoc checks.

## `layout(data_type)` is the machine-readable storage policy

The function `layout(data_type)` returns a `DataTypeLayout`, including `BufferSpec`s describing the expected buffers.

This is where Arrow's physical rules are turned into code that other layers can consume. If you want to know "what buffers should this type have?", this function is one of the best anchors in the codebase.

## Why `ArrayData` is kept generic instead of using only typed arrays

Three reasons:

1. IPC, FFI, and generic readers need one universal representation.
2. Validation logic should not be duplicated across every typed array.
3. Nested and dynamically typed processing often needs to reason about layout before it knows the exact concrete Rust array type.

So `ArrayData` is not just a fallback abstraction. It is the canonical physical contract.

## Source markers

- `ArrayData` definition and memory layout docs:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L205)
- Canonical buffer creation per datatype:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L59)
- Checked vs unchecked construction:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L273),
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L315)
- Validation entry point:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1271)
- Physical layout descriptor:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1690),
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1886)
- Builder for assembling `ArrayData`:
  [`arrow-data/src/data.rs`](../../arrow-data/src/data.rs#L1984)
