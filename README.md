# deflog-parser

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The format-string grammar of deferred-format logging: `{}`, `{=u8}` and
the other type hints, `{:x}` and the other display hints, `{=?}`,
`{=[u8]}`, bitfields like `{0=0..4}`, escaped braces, and positional or
implicit indices — parsed into a typed fragment list, with refusals that
name the byte and the reason.

A `Str` in, a list out.  The parse allocates the list and the literal
strings in it, and nothing else.

## The name, and the wire

The wire format, the linker section names, the tags and this grammar are
**`defmt`'s** (MIT/Apache), unchanged.  That is deliberate and it is the
recommendation the whole deferred-logging story rests on: a novo-lang
firmware that speaks defmt's wire is decoded on day one by `probe-rs
run`, by `defmt-print`, and by every viewer that already exists, and
this decoder reads a Rust firmware for the same reason.

The **name** is the novo-lang family's.  The must-have plan's decision
36 gives a port the upstream name with `-nv`; these two packages take
plain names because `deflog` is the family the device-side encoder,
transports and test harness all belong to, and a family whose members
were `defmt-parser-nv` and `deflog-rtt` would be two families.  So:
`deflog-parser` ports `defmt-parser`, the bytes are byte-for-byte
compatible, and the README says so rather than the package name.

Measured against **`defmt-parser` 1.0**.  Its test module is the oracle
for `tests/fmtparse_tests.nv`.

## Adding it, and checking it

```bash
novo pkg add deflog-parser    # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/fmtparse_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: deflog-parser.<module>.<fn>`.

## The one example that will work

One format string, and the fragment list it becomes.

```novo
use fmtparse
use fmtspec

fn main() [io]
    let frags = fmtparse.parse("x = {=u8:#x} ({0=0..4})")!
    for f in frags
        match f
            FragLiteral(t) => println("literal ${t}")
            FragParam(p)   => println("arg ${p.index} as ${fmtspec.type_token(p.kind)}")
```

```
literal x =
arg 0 as u8
literal  (
arg 0 as 0..4
literal )
```

Three literals and two parameters, both reading argument 0: one printing
the whole byte in hexadecimal, one printing its low four bits.  The
argument is on the wire **once** — which is only expressible because
every index that comes out of a parse is resolved, whether the string
wrote `{}` or `{0}`.

## The layer, and why

`core`.  A format string is a `Str` the caller already holds — out of an
ELF symbol name, or out of a literal at a call site — and parsing it is
arithmetic over its bytes.  Not one function here has an effect row.

There is deliberately **no feed-and-drain reader**, and the absence is
the design.  csv-nv streams because a CSV file does not fit in memory; a
format string is forty characters and arrives whole.  A package that
offered to parse one a chunk at a time would be offering a shape nothing
in this story can use.

**There is no `tests/embedded_probe.nv`, and its absence is a filed
defect rather than a decision.**  The fragment walk is the one part of
this story a device might genuinely want — a firmware that filters its
own log lines by level before encoding them reads the same tags — so a
probe would have been the honest claim.  It cannot be made:
`Result<T, E>` cannot be spelled at `@tier(embedded)`, because with the
package's own error type the compiler refuses `impl Error for
DeflogParseError` (the `Error` trait is not in the prelude at that tier,
E2005) and without the impl it refuses the `Result` itself (SPEC § 3.4,
E2018).  That is
`result-is-unusable-at-tier-embedded-no-error-trait`, open against the
toolchain.  `parse` keeps its `Result` rather than retreating to `?T` to
make a probe build: a package that dropped its error reporting to pass
an audit row would be reporting a toolchain defect as a design.

The `core-embedded` audit row passes anyway — a `core` package with no
probe makes no claim — and this paragraph is the disclosure that one was
wanted.

## The load-bearing interface

Two things, and the second is why the first has to be right.

```novo
pub enum DeflogArgType
    ArgU8
    ArgU16
    ArgU24
    // …
pub fn arg_types(fragments: [DeflogFragment]) -> Result<[DeflogArgType], DeflogParseError>
```

**The type hint is the schema.**  The device sends raw bytes with no
tags in them: four for a `u32`, one for a `u8`, a LEB128 varint for a
`usize`, a length and then bytes for a `str`.  *Nothing on the wire says
which.*  The only thing that can tell a decoder how many bytes to take
and how to read them is the type hint in the format string, which is in
the ELF.  So this is not decoration — it is the schema, and parsing it
is a prerequisite to reading a single argument byte.

That is why an unknown **type** hint is refused in both modes while an
unknown **display** hint is refused only in `ModeStrict`.  A wrong type
desynchronises the stream and turns every later line into garbage; a
wrong display prints a decimal number where somebody wanted hex.
Different severities, so `DeflogParam` carries them in different fields
and `fmterr.is_type_fault` is the predicate that separates them.

`arg_types` is the second, and it is not `params` with the parameters
sorted:

- several holes may read one argument — `{0} {0:x}` does, and a bitfield
  always does;
- an index may be mentioned *only* through bitfields, in which case its
  width is the highest bit any of them reads;
- a hole that reads bits contributes no bytes of its own, which is why
  `fmtspec.arg_width` of a bitfield is `Some(0)` and not `None`.

Resolving all of that into one type per index is the whole job, and
doing it here means a decoder never does it twice and never does it
differently.

## What this does not do, on purpose

- **It does not read arguments.**  It has no bytes; it has a string.
  `deflog-decoder` applies these fragments to a frame.
- **It does not render.**  `fmtspec.hint_token` says how a hint is
  *spelled*; what `:x` produces for a given value is the decoder's, and
  a package that rendered would have had to decide what a `{=?}` looks
  like without a symbol table to look it up in.
- **It does not know about ELF, varints or framing.**  Nothing here
  depends on anything.  That is what lets a compiler link this to check
  a call site without taking a decoder's dependency closure with it —
  which is the reason the grammar is a separate package from the decoder
  at all, upstream and here.
- **It does not resolve `{=istr}`.**  An interned string argument is an
  index into a table this package has never seen.

## The reference implementation

`defmt-parser` 1.0 (MIT/Apache).  Its `Fragment`, `Parameter`, `Type`
and `DisplayHint` are the shapes here, and its test module is the
oracle.

Four things change in the port.

`Type::U8` and its siblings become `ArgU8` and so on: enum variants are
constructed by bare name across a whole assembly, so `U8`, `Str`,
`Debug` and `Bool` as variant names belong to whichever package declared
them first, and a program holding this decoder and anything else would
have two.  `docs/publishing.md` § Public type names are globally unique
is the rule; `Arg…`, `Hint…`, `Frag…`, `Fmt…` and `Log…` are this
package's prefixes.  The same reason gives `LogInfo` rather than `Info`
for a level.

`Cow<'f, str>` on a literal becomes `Str`.  novo-lang has no borrowing
string, so a literal is a copy; the strings are short and there is one
list of them per format string, not per line decoded.

`ParserMode::{Strict, ForwardsCompatible}` becomes `DeflogMode` with the
same two arms and the same meaning, spelled `ModeStrict` and
`ModeForwardCompatible` for the reason above.

And `Level` moves *in* rather than out: upstream keeps it in the parser
crate because a log line's level is decided when its format string is
interned, and the decoder takes it from there.  `deflevel` is that,
including `level_tag`, which is the linker section name — `defmt_info`
and its four siblings, unchanged, because that spelling is the
compatibility.

## Status

| item | implemented |
| --- | --- |
| `deflevel` — `DeflogLevel` | type only |
| `deflevel.level_token`, `.level_of_token`, `.level_rank`, `.level_allows`, `.level_tag`, `.level_of_tag` | no |
| `fmtspec` — `DeflogArgType`, `DeflogPrecision`, `DeflogHint`, `DeflogParam`, `DeflogFragment`, `DeflogMode` | types only |
| `fmtspec.type_token`, `.type_of_token`, `.hint_token`, `.hint_of_token` | no |
| `fmtspec.arg_width`, `.is_bitfield`, `.is_nested`, `.is_signed` | no |
| `fmtparse.parse`, `.parse_with`, `.check` | no |
| `fmtparse.params`, `.arg_types`, `.param_count`, `.literals` | no |
| `fmtparse.max_bitfield_end`, `.has_nested`, `.to_format` | no |
| `fmterr` — `DeflogParseError` | type only |
| `fmterr.offset_of`, `.is_type_fault`, and the `Error` impl | no |
