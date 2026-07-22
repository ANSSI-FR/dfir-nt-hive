# REGF 1.3 Large-Value Parsing Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Parse external values larger than 16,344 bytes as contiguous cells in legacy REGF hives while preserving segmented big-data parsing in REGF 1.5 and newer hives.

**Architecture:** Add one private, version-aware predicate to `KeyValue` and use it in both data dispatch and signature validation. Exercise the real 18,890-byte `ProgramsCache` value from the supplied REGF 1.3 hive, while retaining the existing REGF 1.5 boundary test.

**Tech Stack:** Rust 2021, Cargo, inline unit tests, zero-copy hive parsing.

## Global Constraints

- Work on branch `fix/regf-v13-large-values` in the current checkout; do not create a worktree.
- Copy `user.2008.dat` unchanged to `testdata/user.2008.dat`; its size is 524,288 bytes and its SHA-256 is `cebfab152bb30a60b0d5b14f2cd70dc3d4da11140db786e98eed2b8785ba57f8`.
- Treat REGF minor versions 0 through 4 as legacy contiguous-value formats.
- Treat external values larger than 16,344 bytes as segmented only for REGF minor version 5 or newer.
- Keep `Cargo.lock`, public APIs, error variants, and unrelated parser behavior unchanged.
- The clean baseline is 14 passing unit tests and 0 failing tests; six pre-existing lifetime-syntax warnings are out of scope.

---

### Task 1: Add and fix the REGF 1.3 large-value regression

**Files:**

- Create: `testdata/user.2008.dat`
- Modify: `src/key_value.rs:16-199`
- Test: `src/key_value.rs:583`

**Interfaces:**

- Consumes: `Hive::minor_version() -> u32`, `HiveMinorVersion::WindowsXP`, `BIG_DATA_SEGMENT_SIZE`, `KeyValue::data()`, and `KeyValue::validate_signature()`.
- Produces: private `KeyValue::uses_big_data(&self, data_size: usize) -> bool`; no public API changes.

- [ ] **Step 1: Copy and verify the authorized regression fixture**

Run:

```bash
cp /home/asalais/dev/dfir-ogre/manual_tests/debug_data/user.2008.dat testdata/user.2008.dat
sha256sum testdata/user.2008.dat
stat -c '%s bytes' testdata/user.2008.dat
```

Expected:

```text
cebfab152bb30a60b0d5b14f2cd70dc3d4da11140db786e98eed2b8785ba57f8  testdata/user.2008.dat
524288 bytes
```

- [ ] **Step 2: Write the failing regression test**

Append this test to `src/key_value.rs`'s existing `tests` module:

```rust
#[test]
fn test_regf_v13_large_value_uses_contiguous_cell() {
    let user_hive = include_bytes!("../testdata/user.2008.dat");
    let hive = Hive::new(&user_hive[..]).unwrap();
    assert_eq!(hive.minor_version(), HiveMinorVersion::WindowsNT4 as u32);

    let key_value = hive
        .root_key_node()
        .unwrap()
        .subpath("Software\\Microsoft\\Windows\\CurrentVersion\\Explorer\\StartPage")
        .unwrap()
        .unwrap()
        .value("ProgramsCache")
        .unwrap()
        .unwrap();

    assert_eq!(key_value.data_type().unwrap(), KeyValueDataType::RegBinary);
    assert_eq!(key_value.data_size(), 18_890);

    match key_value.data().unwrap() {
        KeyValueData::Small(data) => assert_eq!(data.len(), 18_890),
        KeyValueData::Big(_) => panic!("REGF 1.3 values must remain contiguous"),
    }

    assert!(key_value.validate_signature().unwrap());
}
```

- [ ] **Step 3: Run the focused test and verify the red failure**

Run:

```bash
cargo test --all-features --locked key_value::tests::test_regf_v13_large_value_uses_contiguous_cell -- --exact --nocapture
```

Expected: FAIL at `key_value.data().unwrap()` because the current parser treats the first bytes of the contiguous 18,890-byte cell as a big-data header with zero segments.

- [ ] **Step 4: Implement the minimal format-aware dispatch**

Change the hive import in `src/key_value.rs` to:

```rust
use crate::hive::{Hive, HiveMinorVersion};
```

Add this private method immediately before `validate_data_signature`:

```rust
fn uses_big_data(&self, data_size: usize) -> bool {
    data_size > BIG_DATA_SEGMENT_SIZE
        && self.hive.minor_version() >= HiveMinorVersion::WindowsXP as u32
}
```

Replace `validate_data_signature` with:

```rust
/// Validates the big-data signature when this hive format uses segmented data.
fn validate_data_signature(&self) -> Result<bool> {
    let header = self.header();

    let data_size_field = header.data_size.get();
    let data_stored_in_data_offset = data_size_field & DATA_STORED_IN_DATA_OFFSET > 0;
    let data_size = (data_size_field & !DATA_STORED_IN_DATA_OFFSET) as usize;
    if !data_stored_in_data_offset && self.uses_big_data(data_size) {
        let cell_range = self
            .hive
            .cell_range_from_data_offset(header.data_offset.get())?;

        return BigDataSlices::has_valid_signature(self.hive, cell_range);
    }
    Ok(true)
}
```

In `KeyValue::data`, replace:

```rust
} else if data_size <= BIG_DATA_SEGMENT_SIZE {
```

with:

```rust
} else if !self.uses_big_data(data_size) {
```

Update the adjacent comments to state that legacy formats may keep larger values in one cell and that segmented big data applies to format 1.5 and newer.

- [ ] **Step 5: Run the regression test and verify green**

Run:

```bash
cargo test --all-features --locked key_value::tests::test_regf_v13_large_value_uses_contiguous_cell -- --exact --nocapture
```

Expected: PASS (`1 passed; 0 failed`).

- [ ] **Step 6: Verify REGF 1.5 segmented data remains supported**

Run:

```bash
cargo test --all-features --locked big_data::tests::test_big_data -- --exact --nocapture
```

Expected: PASS (`1 passed; 0 failed`), including the 16,345-byte `KeyValueData::Big` assertion.

- [ ] **Step 7: Format and run the complete verification suite**

Run:

```bash
cargo fmt --check
git diff --check
cargo test --all-features --locked
```

Expected: formatting and diff checks produce no output; all 15 unit tests pass, with only the six pre-existing lifetime-syntax warnings.

- [ ] **Step 8: Commit the implementation**

```bash
git add src/key_value.rs testdata/user.2008.dat
git commit -m "fix: parse large values in legacy hives"
```
