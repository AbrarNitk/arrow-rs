# 9. Row Format, SIMD, And Execution-Oriented Patterns

Arrow is fundamentally columnar, but real execution engines still need fast row-wise comparisons for:

- lexicographic sort,
- grouping,
- hash and window preparation,
- merge-like operators.

That is why `arrow-rs` has a separate `arrow-row` crate.

## Why a row format exists in a columnar system

Columnar memory is ideal for:

- scanning one column,
- vectorized arithmetic,
- filter masks,
- projection.

But comparing multi-column tuples repeatedly can be awkward if every comparison has to hop across many arrays and type-specific comparison rules.

`arrow-row` solves this by converting columns into normalized row bytes that compare correctly with `memcmp`.

## The conversion idea

```text
Input columns
  col0: UInt64
  col1: Utf8
  col2: Float64

        │
        ▼
RowConverter

        ▼
row0 -> [normalized bytes...]
row1 -> [normalized bytes...]
row2 -> [normalized bytes...]
```

After conversion, row comparison can often be reduced to byte comparison.

## Why this is fast

The row docs explain that rows are normalized for sorting. That means the encoding is chosen so that binary order matches logical Arrow order.

This unlocks:

- `memcmp`-based comparison,
- radix-sort-friendly behavior,
- more predictable branching than repeated per-column dynamic dispatch.

So Arrow stays columnar for storage and scan-heavy compute, but introduces a row-oriented derivative format when comparison-heavy algorithms benefit from it.

## Encoding strategy

The crate docs explain the normalization rules for:

- unsigned integers,
- signed integers,
- floats,
- fixed-length bytes,
- variable-length bytes.

For signed integers, for example, the sign bit is flipped before encoding so that lexicographic byte order matches numeric order.

That is the important pattern: encode values into a sortable byte domain.

## Dictionary flattening is a deliberate tradeoff

The docs note that dictionary arrays are flattened to underlying values during row conversion. That may lose the original dictionary type on the way back out, but it avoids carrying type-specific dictionary comparison complexity in the row format.

This is a classic execution tradeoff:

- preserve exact original representation, or
- normalize aggressively for algorithmic speed.

`arrow-row` chooses the second.

## Where SIMD fits in the story

The repo does not centralize "SIMD" into one magic module because Arrow's performance story is broader:

- aligned contiguous buffers help vector loads,
- bit-packed null buffers help dense validity handling,
- pointer layout choices help LLVM vectorization,
- batch-oriented execution amortizes overhead.

The top-level Arrow README also calls out `target-cpu=native` and AVX512-related performance guidance. That is another clue that the implementation is designed to let the compiler and CPU exploit the physical layout aggressively.

## Source markers

- Comparable row format overview and normalization docs:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L1)
- `RowConverter`:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L532)
- Pointer-layout choice that helps vectorization:
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs#L71)
- Cache-aware alignment policy:
  [`arrow-buffer/src/alloc/alignment.rs`](../../arrow-buffer/src/alloc/alignment.rs#L17)
- Top-level performance guidance:
  [`arrow/README.md`](../../arrow/README.md)
