# sfv-nv

Structured Field Values for HTTP, specified in
[RFC 9651](https://www.rfc-editor.org/rfc/rfc9651) (which replaces RFC 8941),
is a way to write the value of an HTTP header or trailer so that every
implementation reads it the same way. A field value is an item, a list or a
dictionary. This package brings it to novo-lang: it reads a field value into a
tree of typed values by the algorithm of RFC 9651 section 4.2, and writes a
tree back in the canonical form of section 4.1. It is the field syntax under
`Signature-Input`, `Signature`, `Accept-Signature` and `Content-Digest`, and
[httpsig-nv](https://novo-lang.org/packages/httpsig-nv) is built on it.

## What it is

A field value is one of three shapes.

- An **Item** is one bare value with parameters: `5`, `text/html;q=0.8`.
- A **List** is a comma-separated sequence of members. A member is an Item or
  an **Inner List**, a parenthesised sequence of Items that has parameters of
  its own: `1, (a b);x=1`.
- A **Dictionary** is a comma-separated sequence of keys, each with a member:
  `a=1, b, c=(x y)`. A key with no `=` has the value true.

**Parameters** are `;key=value` pairs after an Item or an Inner List. A
parameter with no `=` has the value true.

A bare value has one of eight types.

| Type | Written | Example |
| --- | --- | --- |
| Integer | up to 15 digits | `-42` |
| Decimal | up to 12 integer digits and 3 fraction digits | `4.5` |
| String | printable ASCII in double quotes | `"hello \"world\""` |
| Token | letters, digits and some marks, starting with a letter or `*` | `text/html` |
| Byte Sequence | base64 between colons | `:aGVsbG8=:` |
| Boolean | `?1` or `?0` | `?1` |
| Date | `@` and seconds since the Unix epoch | `@1659578233` |
| Display String | `%"` and percent-encoded UTF-8 | `%"f%c3%bc%c3%bc"` |

The Date and Display String types are new in RFC 9651. A Decimal here is held
as an integer number of thousandths, so `4.5` is `4500`, and every Decimal
compares and writes exactly.

## Install

```
novo pkg add sfv-nv
```

## Example

This program reads a `Signature-Input`-style dictionary, looks a member up,
and writes the dictionary back.

```novo
use std.list
use sfvparse
use sfvtypes
use sfvwrite

fn main() [io]
    let field = "sig1=(\"@method\" \"@authority\");created=1618884473;keyid=\"test-key\""
    match sfvparse.parse_dictionary(field)
        Err(e) => println(e.message())
        Ok(entries) =>
            match sfvtypes.dict_get(entries, "sig1")
                Some(SfvMemberInner(inner)) =>
                    println("covered components: ${list.len(inner.items)}")
                    let created = sfvtypes.param_get(inner.params, "created") ?? SfvBoolean(false)
                    println("created: ${sfvtypes.as_int(created) ?? 0}")
                _ => println("sig1 is not an inner list")
            match sfvwrite.serialize_dictionary(entries)
                Ok(text) => println(text)
                Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `sfvtypes` | The value types, the error codes, constructors, lookups, and the conversion of a `Float` or a decimal string to a Decimal with the rounding of section 4.1.5. |
| `sfvparse` | `parse_item`, `parse_list` and `parse_dictionary` over a field value, and `join_lines` for a field that arrived as several lines. |
| `sfvwrite` | `serialize_item`, `serialize_list` and `serialize_dictionary`. |
| `sfvchars` | The character classes of the grammar, and checks for a key, a Token, a String and UTF-8, for a caller that wants to know whether a value will write. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/sfv-nv) is that reference.

## The rules a user needs

1. **A parse either succeeds whole or fails whole.** RFC 9651 section 4.2 says
   that a parser which fails ignores the field. A parse here answers an
   `Err(SfvError)` with a code from the `SFV_ERR_` constants and the byte
   offset parsing reached. It never answers part of a value.
2. **Choose the entry point from the field's type.** The type of a field comes
   from the specification that defines the field, not from the text. `5` is
   an Item and also a one-member List, and a parser cannot tell them apart.
3. **Several lines of one field are one field.** HTTP lets a sender split a
   list across field lines. `join_lines` joins them with a comma and a space,
   as RFC 9110 section 5.3 does, and the result goes to the parser. Joining
   the lines of an Item field makes it fail, as the RFC intends.
4. **An empty List or Dictionary is no field.** The empty input parses to an
   empty List or Dictionary. An empty one serialises to the empty string, and
   section 4.1 says to leave the field out rather than send it empty.
5. **Leading and trailing spaces are dropped, tabs are not.** Between the
   members of a List or Dictionary a space or a tab may stand beside a comma.
   Elsewhere only a space is accepted.
6. **A repeated key keeps its first place and its last value.** This holds for
   Dictionary keys and Parameter keys (sections 4.2.2 and 4.2.3.2). A key must
   be lower case; `A=1` is refused.
7. **A number has at most 15 digits.** An Integer is 15 digits at most. A
   Decimal has 12 integer digits and 3 fraction digits at most, and `1.` is
   refused. A value beyond that range is not written: `serialize_*` answers
   `Err` with `SFV_ERR_UNWRITABLE`.
8. **A Decimal is written rounded half to even at the third digit.** `0.0015`
   becomes `0.002` and `0.0025` becomes `0.002`. `decimal_from_float` reads a
   `Float` as the shortest decimal string that gives the same double, so
   `0.5015` is the decimal 0.5015 and writes as `0.502`, not as the binary
   value 0.50149999999999994 would.
9. **A Decimal is written with at least one fraction digit and no trailing
   zeros beyond it.** `2.5`, `2.0`, `0.001`.
10. **A Byte Sequence is read without checking its padding.** Section 4.2.7
    asks a parser not to fail on missing `=` padding or on non-zero bits in
    the last group, and this one does neither. Up to two `=` may end the text.
    A `=` elsewhere, or a length no encoding gives, is refused. It is always
    written with padding.
11. **A String is ASCII.** A byte outside printable ASCII, or an escape other
    than `\"` and `\\`, is refused when reading, and a String with such a byte
    is not written. Text that is not ASCII is a Display String.
12. **A Display String is UTF-8.** It is read from `%"..."` with each `%`
    followed by two lower-case hexadecimal digits, and the decoded bytes
    must be well-formed UTF-8: no overlong form, no surrogate, nothing above
    `U+10FFFF`. Upper-case hexadecimal is refused when reading. It is written
    with lower-case hexadecimal for each byte that is not printable ASCII, and
    for `%` and `"`.
13. **A field value must be ASCII.** A byte above 127 anywhere in the input is
    `SFV_ERR_ASCII`.

## What is not included

- **Field names and the HTTP message.** The package reads and writes the value
  of a field. Which field carries which type, and how a message is framed, are
  the caller's.
- **Allocation-free parsing on a microcontroller.** A parsed Dictionary is a
  growable list of values that hold strings, so no module here builds for the
  embedded tier. The registry shows the tiers each module checks clean at.
- **The Structured Headers draft before RFC 8941.** The older syntax is not
  read.
- **Typed accessors beyond the lookup functions.** A caller matches an
  `SfvBare` itself or uses `as_int`, `as_string` and the other accessors.

## Related packages

- [httpsig-nv](https://novo-lang.org/packages/httpsig-nv) signs and verifies
  HTTP messages by RFC 9421 and reads and writes the `Signature-Input`,
  `Signature` and `Accept-Signature` fields through this package.
- [cookie-nv](https://novo-lang.org/packages/cookie-nv) reads and writes the
  `Cookie` and `Set-Cookie` fields, which are not structured fields.

## Tests

The suite is two files under `tests/`.

`tests/sfv_corpus_tests.nv` runs the HTTP Working Group's
[structured-field-tests](https://github.com/httpwg/structured-field-tests)
corpus, vendored under `tests/data/` at the commit in
`tests/data/SOURCE.txt`. It runs every record of every file: 20 files and
2,160 records. A record that must fail is checked to fail, and any other
record is checked to parse to the expected value and to write back to its
canonical form. The 553 records of `serialisation-tests`, which are values
to write, some of which must be refused, are checked the same way. Eight
records are marked `can_fail`; all eight are accepted and give the expected
value. The corpus includes the field examples of RFC 9651 section 3.

`tests/sfv_api_tests.nv` checks what a corpus cannot say: each error code and
its offset, the character classes, UTF-8 validation at its boundaries,
rounding ties, the refusals of the writer, and the lookup functions.

## Licence

Apache-2.0. See `LICENSE`. The files under `tests/data/` are the
structured-field-tests corpus and carry their own licence, in
`tests/data/LICENSE.md`.
