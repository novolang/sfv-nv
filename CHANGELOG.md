# Changelog

All notable changes to sfv-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`sfvtypes`**: the eight bare types of RFC 9651 section 3.3 as `SfvBare`,
  `SfvItem`, `SfvInner`, `SfvMember` and `SfvEntry`; the `SFV_ERR_` codes and
  `SfvError`; constructors, `param_set` and `dict_set` with the
  first-position, last-value rule, lookups, and `as_*` accessors; and
  `decimal_from_text` and `decimal_from_float`, which round half to even at
  the third fraction digit as section 4.1.5 asks. A Decimal is an integer
  number of thousandths.
- **`sfvparse`**: `parse_item`, `parse_list` and `parse_dictionary` following
  the algorithms of section 4.2, and `join_lines`. The parser keeps one
  cursor and one error for the whole parse.
- **`sfvwrite`**: `serialize_item`, `serialize_list` and
  `serialize_dictionary` in the canonical forms of section 4.1, refusing a
  value that has none with `SFV_ERR_UNWRITABLE`.
- **`sfvchars`**: the character classes of the grammar, and `valid_key`,
  `valid_token`, `valid_string` and `valid_utf8`.
- Two test suites. `sfv_corpus_tests.nv` runs all 2,160 records of the
  httpwg structured-field-tests corpus, vendored under `tests/data/`.
  `sfv_api_tests.nv` covers the API, the error codes, and the edges of the
  character classes, rounding and UTF-8 validation.

### Not included

- Parsing or building on a microcontroller. No module checks clean at the
  embedded tier.
- Field names, and the framing of a message.
- RFC 8941 draft syntax earlier than the published RFC.
