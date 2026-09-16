# deflog-parser

**Deferred-format logging** is a way of logging from a microcontroller in
which the device sends a small number and the raw bytes of the arguments, and
the machine watching holds the text. This package is the grammar of the
format strings that text is written in. A string goes in and a list of typed
fragments comes out, with a refusal that names the byte and the reason. The
grammar is [defmt](https://defmt.ferrous-systems.com/)'s, unchanged, and this
package is measured against `defmt-parser` 1.0. The decoder that applies
these fragments to a frame is
[deflog-decoder](https://novo-lang.org/packages/deflog-decoder).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What a format string is here

A format string is a line of text with **holes** in it. `x = {=u8}` is one
literal, `x = `, and one hole. Everything outside a hole is copied through. A
literal brace is written twice: `{{` prints one `{`.

A hole has up to three parts, in this order.

An **index** says which argument the hole reads. It is written as a number,
`{0}`. A hole with no number takes the next argument that has not been used,
which is called an implicit index. A format string uses one style or the
other and never both.

A **type hint** says how the argument's bytes are to be read. It is written
after `=`: `{=u8}`, `{=i32}`, `{=str}`, `{=[u8]}`, `{=?}`. A **bitfield** is
a type hint written as a range of bit positions, `{0=0..4}`, which reads the
low four bits of argument 0.

A **display hint** says how the value that was read is to be printed. It is
written after `:`: `{=u8:x}` for hexadecimal, `{=u8:#x}` with the `0x`
prefix, `{=u8:04}` padded to four digits, `{=[u8]:a}` as ASCII.

The type hint is not decoration. It is the schema. The device sends raw bytes
with nothing in them that says what they are: four bytes for a 32-bit value,
one for a byte, a variable-length integer for a pointer-sized value, a length
and then the bytes for a string. The only thing that can tell a decoder how
many bytes to take, and how to read them, is the type hint in the format
string. Parsing this grammar is therefore a prerequisite to reading a single
argument byte.

A **level** is what a log line was logged at, and it is fixed when the format
string is interned rather than sent with the line. This package carries the
five levels and the linker section name each one uses.

| Quantity | Value |
| --- | --- |
| Type hints | 29 |
| Display hints | 13 |
| Levels | 5 |
| Reasons a parse is refused | 14 |
| Dependencies | 0 |
| Bytes a bit-field hole contributes to the wire | 0 |

## Install

```
novo pkg add deflog-parser
```

## Example

```novo
use fmtparse
use fmtspec

fn main() [io]
    // One format string, as a firmware would intern it. Two holes read the
    // same argument: the whole byte, and its low four bits.
    match fmtparse.parse("x = {=u8:#x} ({0=0..4})")
        Err(e) => println(e.message())
        Ok(frags) =>
            for f in frags
                match f
                    // The text between the holes, exactly as written.
                    FragLiteral(t) => println("literal ${t}")
                    // The hole: which argument it reads, and how to read it.
                    FragParam(p)   => println("arg ${p.index} as ${fmtspec.type_token(p.kind)}")

            // One type per argument index, however many holes read it.
            match fmtparse.arg_types(frags)
                Err(e)  => println(e.message())
                Ok(ts)  => println("${ts.len()} argument(s) on the wire")
```

Build and test with `novo pkg build` and
`novo test --isolate tests/fmtparse_tests.nv`. Today `novo test` fails on
purpose: every test reaches a `not implemented: deflog-parser.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `fmtparse` | The parse itself, in either mode, and the questions asked of a fragment list: the parameters, one type per argument index, the literals, the widest bit-field read of an index, and the format string written back out. |
| `fmtspec` | The vocabulary: the type hints, the display hints, the precision of a timestamp hint, a hole as a value, a fragment as a literal or a hole, and the two parse modes. It also answers how many bytes a type takes on the wire. |
| `fmterr` | The fourteen reasons a parse is refused, the byte offset of each, and the predicate that separates a fault about a type from a fault about a display. |
| `deflevel` | The five levels, their names, their order, the filter test, and the linker section name each one is interned under. |

## How to choose an entry point

**`fmtparse.parse` is the ordinary call.** It parses in the
forward-compatible mode, which is the one a decoder wants.

**`fmtparse.parse_with` chooses the mode.** The strict mode refuses a display
hint it does not know. Use it in a compiler or a lint that is checking a call
site the author can still fix.

**`fmtparse.check` answers a refusal or nothing, and builds no list.** Use it
where only validity matters.

**`fmtparse.arg_types` is what a decoder reads the wire with.** It is not the
parameter list sorted. It resolves every hole down to one type per argument
index, which is a different thing: several holes may read one argument, and
an argument may be mentioned only through bit-fields.

## The rules a user needs

1. **An unknown type hint is refused in both modes; an unknown display hint
   is refused only in the strict mode.** A wrong type desynchronises the
   stream and turns every later line into nonsense. A wrong display prints a
   decimal number where somebody wanted hexadecimal. `fmterr.is_type_fault`
   is the predicate that tells the two apart.
2. **Every index that comes out of a parse is resolved.** A hole written `{}`
   and a hole written `{0}` both answer a number. A caller never has to track
   which argument is next.
3. **A format string uses positional indices or implicit ones, never both.**
   Mixing them is `FmtMixedIndexing`.
4. **Several holes may read one argument, and the argument is on the wire
   once.** `{0} {0:x}` prints the same value twice from one set of bytes.
   This is why `fmtparse.arg_types` exists and why counting holes is not
   counting arguments.
5. **A bit-field hole contributes no bytes of its own.**
   `fmtspec.arg_width` of a bit-field answers zero rather than nothing. An
   index mentioned only through bit-fields still has a width: it is wide
   enough for the highest bit any of its holes reads, which is what
   `fmtparse.max_bitfield_end` answers.
6. **Two bit-field holes on one index may not overlap, and a bit-field may
   not be mixed with an ordinary type hint on the same index.** Both are
   refusals, `FmtBitfieldOverlap` and `FmtBitfieldMixedWithType`.
7. **Two holes that give one index different types is `FmtTypeConflict`, and
   it names both.**
8. **Every refusal carries the byte offset it happened at.**
   `fmterr.offset_of` reads it, so a compiler can underline the character in
   the call site.
9. **A level is fixed when the format string is interned, not sent with the
   line.** `deflevel.level_tag` is the linker section name it is interned
   under, and that spelling is the compatibility with existing viewers.
10. **The type names in this package carry a prefix, and the prefix is not
    decoration.** A variant is constructed by its bare name across a whole
    assembly, so `U8`, `Str`, `Debug` and `Bool` as variant names would
    collide with another package's. The prefixes here are `Arg`, `Hint`,
    `Frag`, `Fmt` and `Log`, which is why a level is `LogInfo`.
11. **An interned string argument is not resolved here.** `{=istr}` names an
    index into a table this package has never seen. A decoder resolves it.

## What is not included

- **Reading arguments.** This package has a string, not bytes.
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) applies
  these fragments to a frame.
- **Rendering.** `fmtspec.hint_token` says how a hint is spelled. What a
  hexadecimal hint produces for a given value is the decoder's business, and
  a package that rendered would have to decide what a nested value looks like
  without a symbol table to look it up in.
- **Anything about ELF files, variable-length integers or framing.** This
  package depends on nothing, which is what lets a compiler link it to check
  a call site without pulling a decoder's dependencies in with it.
- **A streaming parser.** A format string is a few dozen characters and
  arrives whole. There is nothing to feed in chunks.
- **A microcontroller build.** The package ships no probe program and does
  not build for a microcontroller with no heap allocator. A device that
  filtered its own log lines by level before encoding them would want the
  fragment walk, and it cannot have it today:
  a `Result` cannot be spelled at the device tier, because the error trait
  its error type must implement is not in the prelude there (E2005), and
  without that implementation the `Result` itself is refused
  (SPEC section 3.4, E2018). That is the open toolchain defect
  `result-is-unusable-at-tier-embedded-no-error-trait`. `fmtparse.parse`
  keeps its `Result` rather than dropping its error reporting to make a
  probe build.

## Related packages

- [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) is the
  other half: a firmware image in, structured log records out. It reads every
  argument byte against the types this package resolves.
- [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv) is the framing the
  frames travel in.
- [logging-nv](https://novo-lang.org/packages/logging-nv) is structured
  logging on a host. Its levels are the same five, and a decoded record
  enters a program through its sinks.
- `std.fmt` in the standard library is novo-lang's own string interpolation,
  which formats on the machine that prints. This grammar exists so that the
  machine that prints and the machine that logged can be different machines.

## Tests

```bash
novo test --isolate tests/fmtparse_tests.nv   # 18 tests: the grammar
novo test --isolate tests/fmtspec_tests.nv    #  7 tests: the vocabulary
novo test --isolate tests/deflevel_tests.nv   #  6 tests: the levels
```

The oracle is `defmt-parser` 1.0's own test module, whose `Fragment`,
`Parameter`, `Type` and `DisplayHint` are the shapes here.

The suite asserts that a literal brace is written twice, that an unmatched
brace is refused at its offset, that implicit and positional indices may not
be mixed, that two holes reading one index resolve to one argument, that a
bit-field contributes no bytes, that overlapping bit-fields are refused, that
an unknown type hint is refused in both modes and an unknown display hint
only in the strict one, and that a fragment list written back out is the
string it was parsed from.

The tests compile today and fail at run, each on the
`not implemented: deflog-parser.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a time
as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `deflevel.DeflogLevel` | declared |
| `deflevel.level_token`, `.level_of_token`, `.level_rank`, `.level_allows`, `.level_tag`, `.level_of_tag` | no |
| `fmtspec.DeflogArgType`, `.DeflogPrecision`, `.DeflogHint`, `.DeflogParam`, `.DeflogFragment`, `.DeflogMode` | declared |
| `fmtspec.type_token`, `.type_of_token`, `.hint_token`, `.hint_of_token` | no |
| `fmtspec.arg_width`, `.is_bitfield`, `.is_nested`, `.is_signed` | no |
| `fmtparse.parse`, `.parse_with`, `.check` | no |
| `fmtparse.params`, `.arg_types`, `.param_count`, `.literals` | no |
| `fmtparse.max_bitfield_end`, `.has_nested`, `.to_format` | no |
| `fmterr.DeflogParseError`, its fourteen arms | declared |
| `fmterr.offset_of`, `.is_type_fault`, and the `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
