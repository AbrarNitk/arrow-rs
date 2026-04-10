# 9. Row Format, SIMD, And Execution-Oriented Patterns

This chapter answers a seemingly paradoxical question:

**If Arrow is a columnar format, why does `arrow-rs` also implement a row-oriented format?**

Because “columnar is best” is only true for some classes of work.

## Start with the execution problem

Columnar layout is excellent for:

- scanning one column,
- vectorized arithmetic,
- filtering with masks,
- projection.

But some operations care about *tuples* rather than individual columns:

- lexicographic sort,
- grouping by multi-column keys,
- merge comparisons,
- window partition ordering.

If every tuple comparison must:

- jump across several arrays,
- branch on several types,
- apply several comparison rules,

then the columnar representation is no longer the easiest execution shape for that operator.

So the question becomes:

**Can we derive a row-oriented byte format from Arrow columns such that bytewise order matches logical tuple order?**

`arrow-row` is the answer.

## Why `memcmp` is the goal

The row format is designed so that comparing rows can often reduce to `memcmp`.

That is powerful because it turns:

- many type-specific comparisons,

into:

- one normalized byte comparison.

This supports:

- efficient multi-column sorting,
- radix-sort-style strategies,
- lower branch overhead,
- better cache locality during comparison-heavy phases.

## What “normalized for sorting” really means

The row encoding is not arbitrary serialization. It is carefully chosen so that lexicographic byte order matches Arrow logical order.

That means different types need different normalization tricks:

- unsigned integers can use big-endian style ordering,
- signed integers need sign-bit manipulation,
- floats need IEEE-aware transformation,
- variable-length bytes need sentinel-safe ordering rules.

This is why the row-format docs spend so much time on encoding rules: correctness depends on order preservation, not just reversibility.

## Why the row format is derivative, not primary

Arrow storage remains columnar because that is still the best general-purpose physical format for many workloads.

The row format is a derived execution format:

- built from columns,
- used for comparison-heavy phases,
- sometimes converted back again.

This is the right architecture because it avoids sacrificing the strong properties of columnar storage just to optimize a narrower class of algorithms.

## Why dictionary flattening is a deliberate tradeoff

`arrow-row` flattens dictionary arrays to underlying values during conversion.

Why accept that?

- keeping dictionary identity in the row format would complicate normalized comparison,
- the row format cares about comparison semantics more than representation preservation,
- the conversion is meant for execution, not for round-tripping exact physical identity.

This is a very clear policy decision:

- optimize for comparison behavior,
- not for preserving every original container choice.

## Where SIMD fits into the broader story

Arrow performance is not “there is one SIMD module somewhere.”

It is the compound result of:

- aligned dense buffers,
- packed bitmaps,
- sliceable shared memory,
- compiler-friendly pointer layout,
- batch-oriented operators,
- and, when needed, derived row encodings that improve comparison patterns.

That is why the top-level README talks about `target-cpu=native` and AVX512-related guidance. The repository is structured so the compiler and CPU have a good chance to do the right thing.

## Why the row format may change across releases

The docs explicitly note that the row encoding may change between releases.

That is a useful boundary to understand:

- Arrow's main memory format is the stable interoperability contract,
- the row format is an internal execution-oriented derivative representation.

So it is free to evolve more aggressively as long as it still serves its execution purpose.

## Reimplementation checklist

To rebuild Arrow-style execution support deeply, you need:

1. a columnar primary format,
2. a derived row encoding whose byte order matches logical order,
3. normalization rules per type family,
4. explicit tradeoffs such as dictionary flattening,
5. an understanding that execution formats can evolve faster than the core memory contract.

## Source markers

- Row-format overview and `memcmp` motivation:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L1)
- `RowConverter`:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L532)
- Normalization rules:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L188)
- Dictionary flattening docs:
  [`arrow-row/src/lib.rs`](../../arrow-row/src/lib.rs#L120)
- Pointer layout helping vectorization:
  [`arrow-buffer/src/buffer/immutable.rs`](../../arrow-buffer/src/buffer/immutable.rs#L71)
- Performance guidance:
  [`arrow/README.md`](../../arrow/README.md)
