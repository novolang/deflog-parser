# Changelog

All notable changes to deflog-parser are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## Unreleased — 2026-10-10

### Fixed

- **The embedded probe links and runs.**  The bare-metal runtime now
  defines `novo_str_char_index`, the symbol every `s[i]` lowers to, so
  `novo build --target=nrf52-qemu tests/embedded_probe.nv` produces an
  executable and the executable prints `display fault`, `refused` and
  `signed` on `mps2-an386`.  The device claim is a binary rather than
  the tier check, and the README and the probe's own header say so now.
  No line under `src/` changed: nothing here worked around the missing
  symbol, so there is nothing to revert.  The 0.1.0 entry below is left
  as written, and its `Known` section is no longer true.

## 0.1.0 — 2026-10-10

**Every body has one.**  The signatures, the effect rows and the doc
comments are what 0.0.3 published; 0.1.0 is the release that makes them
run.  The four modules check clean on a host and at the embedded tier,
the three API suites and the new edge suite pass, and the four doc
examples run.  The embedded probe stops at the link, and why is under
Known.

### Added

- `tests/deflog_parser_edge_tests.nv` — the edges the three API suites
  leave alone: every spelling in the type and display-hint grammars
  read back through its own token, every one of the fourteen refusals
  with the byte it points at and whether it is a fault about the wire,
  and both spellings of each level.  The four suites together reach
  every one of the 622 lines `src/` answers to, and no line carries a
  skip marker.

### Changed

- `fmtparse.check` shares the parse and drops the list it built.  The
  README said it built no list; the doc comment, which is the published
  contract, says "the same work, a different return", and that is what
  it is.
- Every cross-module call in `fmtparse` is now written `fmtspec.…`.
  The qualification is what keeps the heap value the call answers from
  being stranded, and it reads no worse.
- Every construction that allocates sits inside a `system` block, which
  is the discharge the embedded tier names: the fragment list, the
  payload-carrying `DeflogFragment` and `DeflogParseError` variants,
  every `Some` of one, and every string concatenation.  The blocks are
  where the tier puts the allocator, and the host builds no differently
  for them.

### Design notes

`parse` is the forward-compatible mode, which the README's entry-point
table already said and the `parse` doc comment only implied: it frames
`ModeStrict` and sends a decoder to `parse_with(s,
ModeForwardCompatible)`.  The README is the published contract, so
`parse` is `parse_with(format, ModeForwardCompatible)`, and a test now
holds it there.

`HintSeqDebug` is in the vocabulary but no token produces it, and
`hint_token` spells it `?` like `HintDebug`.  That is upstream's shape
too, and it is recorded here because it means the hint tokens are not a
bijection: a round trip through a spelling is asserted only for the
hints a parse can produce.

A `__internal_bitflags_…` hint with four `@`-separated parts is not a
shape this grammar has.  Upstream's wire format version 4 added the
fourth part, so it is kept whole by `ModeForwardCompatible` and refused
by `ModeStrict`, exactly like any other hint this package cannot read.

### Known

- The embedded tier stamps, checks and lowers the four modules for
  `nrf52-qemu`, and the probe then stops at the link: the bare-metal
  runtime defines no `novo_str_char_index`, and every substring here is
  copied one byte at a time with the index sugar that needs it.  The
  device claim is the tier check and not an executable, so it is
  unverified past the check.  The defect is
  `index-sugar-unlinkable-at-tier-embedded-no-novo-str-char-index`; the
  shape the package keeps instead of working around it is in the
  findings below.

### Toolchain findings

Recorded because each one shaped the implementation, and each is a
defect rather than a language decision.

- `s[lo..hi]` on a `Str` compiles on every target and panics at run
  with a nonsense index, and the embedded checker admits it although it
  refuses `str.slice`.  The emitted IR says why: the slice sugar builds
  a list for the range and then calls `novo_str_char_index(s,
  ptrtoint(range_list))`, so the index it passes is a heap pointer.  No
  substring in this package is spelled as a range.
- `s[i]` reads one byte as a one-byte `Str` and lowers to
  `novo_str_char_index`, which the host runtime defines and the
  bare-metal runtime does not.  The embedded checker admits the sugar,
  so the check passes and the link is what fails — the shape of defect
  0.8.6 fixed by defining `novo_str_byte_at`.  The entry that would do
  the same job by name, `str.from_byte`, is refused at that tier even
  though the runtime does define its symbol, so the sugar is the only
  route to a one-byte `Str` on a device, and this package takes it.
- A `system` block's value is discarded: the block runs, but its last
  expression reads as the zero value.  Every allocating function here
  returns from inside the block or through a variable declared outside
  it.  Filed as `a-system-block-in-value-position-discards-its-value`,
  fixed in 0.20.2 for a function's own tail; the workaround stays until
  `a-system-block-whose-value-is-a-result-call-is-refused-at-a-device-boundary`
  closes.  A sweep of all twenty sites here was tried on 2026-10-10 and
  undone: the device build refused six of them, every one a function
  answering `Result<…>`.
- A one-byte `Str` made by the index sugar in an operand position is
  stranded when it is bound to a local, and a call into a sibling
  module that is not qualified with the module name strands the heap
  value it answers.  Both showed up as leaked objects and a non-zero
  exit from every green suite.  One function, `fmtspec.one_byte`, is
  the only place a byte becomes a string, and every cross-module call
  is qualified.

## [0.0.3] — 2026-10-06

No signature, type or effect row changed, and every body is still
`todo()`.

### Added

- `tests/embedded_probe.nv`, a program that builds the four modules for
  a Cortex-M4 (`novo build --target=nrf52-qemu`).  It parses a format
  string, matches the `Result`, and asks a level and a type the
  questions a device asks before encoding a line.

### Changed

- The README has a "Running on a microcontroller" section and no longer
  lists a microcontroller build under what is not included.  Version
  0.0.2's README said the package did not build for a device, because a
  `Result` could not be spelled at the embedded tier: the `Error` trait
  was not in the prelude there (E2005), so the `Result` itself was
  refused (E2018).  That was the toolchain defect
  `result-is-unusable-at-tier-embedded-no-error-trait`, fixed in novo
  0.9.0.  The registry's facets for 0.0.2 already listed the embedded
  tier, and the README now agrees with them.
- The example carries the interface stamp beside it.

## [0.0.2] — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

### Design notes

What changes in the port from `defmt-parser` 1.0, recorded here because
the README no longer carries it.

`Type::U8` and its siblings become `ArgU8` and so on.  An enum variant
is constructed by its bare name across a whole assembly, so `U8`,
`Str`, `Debug` and `Bool` as variant names would belong to whichever
package declared them first, and a program holding this package and
anything else would have two.  `Arg`, `Hint`, `Frag`, `Fmt` and `Log`
are this package's prefixes, which is also why a level is `LogInfo`.

`Cow<'f, str>` on a literal becomes `Str`.  novo-lang has no borrowing
string, so a literal is a copy; the strings are short and there is one
list of them per format string, not per line decoded.

`ParserMode::{Strict, ForwardsCompatible}` becomes `DeflogMode` with the
same two arms and the same meaning, spelled `ModeStrict` and
`ModeForwardCompatible`.

`Level` moves in rather than out.  Upstream keeps it in the parser crate
because a log line's level is decided when its format string is
interned, and the decoder takes it from there.  `deflevel` is that,
including `level_tag`, the linker section name, unchanged because that
spelling is the compatibility.

The name is the novo-lang family's rather than the upstream name with a
suffix.  `deflog` is the family the device-side encoder, the transports
and the test harness all belong to, and a family whose members were
`defmt-parser-nv` and `deflog-rtt` would be two families.
