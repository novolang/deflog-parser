# Changelog

All notable changes to deflog-parser are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-10

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `fmtspec` — the grammar's vocabulary. `DeflogArgType` is the wire
  type of an argument, which is the schema the whole deferred-format
  model rests on: the device sends bytes with no tags in them, and this
  hint is the only thing that says how many to take. `DeflogHint` is
  how the value is displayed once read, and the two are separate fields
  because a wrong type desynchronises the stream while a wrong hint
  prints a decimal number.
- `fmtparse` — `parse`, `parse_with` and `check`, and the three queries
  a consumer actually asks: `arg_types` (one type per argument, in wire
  order, with several holes reading one argument already resolved),
  `param_count` (what a compiler compares against a call site) and
  `max_bitfield_end` (what decides how wide a bitfield argument is
  sent). There is deliberately no feed-and-drain reader: a format
  string arrives whole, and a chunk-at-a-time parser would be a shape
  nothing in this story can use.
- `fmterr` — fifteen reasons, each carrying the byte offset in the
  format string. `is_type_fault` separates the refusals that make a
  firmware undecodable from the ones that only affect how it looks,
  which is what lets a host decode what it can rather than refusing a
  whole file.
- `deflevel` — the five levels, and `level_tag`, the linker section
  name each is interned under. The level lives here rather than in the
  decoder because it is decided when the format string is interned, and
  because `defmt-parser` owns it upstream for the same reason.

### Compatibility

- The grammar, the tags and the linker section names are `defmt`'s
  (MIT/Apache), unchanged and measured against `defmt-parser` 1.0, so
  that a novo-lang firmware decodes under `probe-rs run` and
  `defmt-print` on day one. The package name is the novo-lang family's;
  the README says why.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  deflog-parser.<module>.<fn>`. The vectors in
  `tests/fmtparse_tests.nv` are `defmt-parser`'s own.
- There is no `tests/embedded_probe.nv`, and its absence is a filed
  defect rather than a decision: the fragment walk is code a device
  might want, and `Result` cannot be spelled at `@tier(embedded)` today
  (`result-is-unusable-at-tier-embedded-no-error-trait`). The
  signatures keep their `Result`.
