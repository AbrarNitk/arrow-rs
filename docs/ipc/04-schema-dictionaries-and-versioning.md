# 4. Schema, Dictionaries, and Versioning

This chapter covers the parts of IPC that are most stateful and easiest to get wrong:

- schema encoding,
- dictionary ids and updates,
- metadata version choices,
- endianness checks.

## Schema Encoding

`IpcSchemaEncoder` in `arrow-ipc/src/convert.rs` is the core schema serializer.

Its job is not just "write fields." It must preserve enough semantics for a remote reader to rebuild Arrow types faithfully:

- nested field structure,
- nullability,
- metadata,
- dictionary encoding,
- target endianness.

That is why schema conversion logic lives in its own module and not inside the record batch writer loop.

## Why Dictionary IDs Exist

Dictionary arrays are stateful across messages.

The schema tells the reader that a field is dictionary-encoded, but the dictionary values themselves may arrive separately in `DictionaryBatch` messages. Therefore the reader needs a stable key to match:

- schema field,
- dictionary value payload,
- later record batch index buffers.

That stable key is `dict_id`.

## `DictionaryTracker`

The writer-side state lives in `DictionaryTracker` in `arrow-ipc/src/writer.rs`.

It answers several questions:

- which dictionary ids have already been assigned?
- which actual dictionary values have already been sent?
- if a dictionary changes, is it a replacement, delta, or no-op?

This tracker is the reason dictionary handling cannot be a pure stateless function of one batch.

## Why Dictionary Traversal Is Depth-First

The writer code comments around `encode_dictionaries` explain an important subtlety: nested dictionaries must consume dictionary ids in depth-first order so child dictionaries are assigned before parent dictionary fields that depend on them structurally.

This matters because nested structures like:

- `List<Dictionary<...>>`
- `Struct<..., Dictionary<...>>`

need deterministic id assignment consistent with schema traversal and encoding order.

## Dictionary Handling Policies

`arrow-ipc/src/writer.rs` exposes dictionary handling policy through `IpcWriteOptions`.

The interesting mode is `DictionaryHandling::Delta`.

Why delta dictionaries exist:

- dictionary-valued streams often evolve incrementally,
- resending the full dictionary every time wastes bandwidth,
- appending only new values keeps keys stable and transfer smaller.

Why they are hard:

- the writer must detect whether the new dictionary really extends the old one,
- the reader must combine old and new values correctly,
- replacement and delta semantics must stay distinct.

The tests in `arrow-ipc/tests/test_delta_dictionary.rs` are worth studying because they exercise nested and multi-type dictionary cases.

## Reader-Side Dictionary Updates

On the reader side, `read_dictionary_impl`, `get_dictionary_values`, and `update_dictionaries` in `arrow-ipc/src/reader.rs` reconstruct dictionary state.

The logic is:

```text
DictionaryBatch
  -> read dictionary value array using the schema field's value type
  -> if not delta: replace cached dict_id entry
  -> if delta: concatenate onto existing cached dict_id entry
```

This is a very important design point:

the dictionary batch itself does not redundantly carry the full value type in a standalone way. The reader uses the schema and dictionary id to infer how to decode the values.

## Metadata Version

`IpcWriteOptions` in `arrow-ipc/src/writer.rs` carries `metadata_version` and `write_legacy_ipc_format`.

Important policy from the code:

- the Rust writer supports V4 and V5 writing,
- default is V5,
- compression requires V5,
- legacy format is only allowed with V4.

This policy makes sense:

- modern behavior should be the default,
- older compatibility paths should remain explicit,
- features introduced later should not be silently backported into older metadata versions.

## Version-Specific Semantics

There are several places where version changes matter:

- continuation marker behavior,
- union/null validity bitmap handling,
- compression availability.

Examples from code:

- `write_continuation` in `writer.rs`
- `has_validity_bitmap` in `writer.rs`
- union decode rules in `reader.rs`

This is why "metadata version" is not just a tag for informational display. It affects decode interpretation.

## Endianness

`Endianness::equals_to_target_endianness` in `arrow-ipc/src/lib.rs` and the file reader checks in `reader.rs` enforce a hard rule:

- the IPC data must match the target system endianness for correct interpretation.

Why this check exists:

- Arrow buffers are low-level typed memory layouts,
- byte-swapping every buffer transparently would complicate zero-copy semantics heavily,
- explicit rejection is safer than silent misinterpretation.

## Source Markers

- `arrow-ipc/src/convert.rs`: schema conversion and dictionary field encoding
- `arrow-ipc/src/writer.rs`: `DictionaryTracker`, dictionary update policy, `IpcWriteOptions`
- `arrow-ipc/src/reader.rs`: dictionary decoding/update logic, version-sensitive decoding
- `arrow-ipc/tests/test_delta_dictionary.rs`: nested and delta dictionary examples
- `arrow-ipc/src/lib.rs`: endianness helper

## Reimplementation Lesson

A correct IPC implementation must treat dictionaries and versioning as stream/file state, not as local batch decoration. That is one of the places where simplistic serializers usually fail.
