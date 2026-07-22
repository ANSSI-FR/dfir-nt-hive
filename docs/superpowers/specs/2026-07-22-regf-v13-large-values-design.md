# REGF 1.3 Large-Value Parsing Design

## Problem

`KeyValue::data` currently treats every external value larger than 16,344 bytes as a segmented big-data value. That rule is valid for REGF format 1.5 and later, but `user.2008.dat` is format 1.3. Its `ProgramsCache` value contains 18,890 bytes directly in one allocated cell, so interpreting the first bytes as a `db` header produces a zero segment count and a parsing error.

## Design

Add one private predicate in `key_value.rs` that determines whether an external value uses the segmented big-data representation. It returns true only when both conditions hold:

- The declared data size is greater than `BIG_DATA_SEGMENT_SIZE` (16,344 bytes).
- The hive minor version is `HiveMinorVersion::WindowsXP` (5) or newer.

`KeyValue::data` will use this predicate to select `KeyValueData::Big`; otherwise it will read the referenced cell as `KeyValueData::Small`. Inline values remain unchanged. REGF minor versions 0 through 4, including the uncommon 1.4 variant, therefore retain the legacy contiguous representation.

`KeyValue::validate_data_signature` will use the same predicate. It will require and validate the `db` signature only for values that the data path treats as segmented. Direct values continue to be bounded by the referenced cell, so malformed sizes still return the existing cell-range errors.

## Regression Coverage

Copy the supplied hive to `testdata/user.2008.dat` without modifying it. Add a unit test that opens the hive, resolves `Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\StartPage`, reads `ProgramsCache`, and verifies that:

- The hive minor version is 3.
- The value is represented as `KeyValueData::Small`.
- Reading the value returns all 18,890 declared bytes.
- Signature validation succeeds.

Keep the existing format 1.5 boundary tests, including the 16,345-byte value represented as `KeyValueData::Big`, to prevent the compatibility fix from disabling segmented data.

## Scope

The patch changes only format-aware dispatch and its matching validation rule. It does not alter the big-data iterator, public APIs, error variants, dependency versions, or unrelated parser behavior.
