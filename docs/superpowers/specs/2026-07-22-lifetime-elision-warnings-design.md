# Lifetime-Elision Warning Cleanup Design

## Problem

Rust 1.95 reports six `mismatched_lifetime_syntaxes` warnings. Each affected method elides the lifetime on `&self` or `&mut self` while also omitting that inferred lifetime from a generic return type. The compiler still infers the intended borrow correctly, but the mixed notation makes the relationship less visible.

Promoting warnings to errors with `cargo rustc --lib --all-features --locked -- -D warnings` reproduces all six diagnostics and no unrelated diagnostic.

## Design

Follow the compiler's minimal recommendation by writing the anonymous placeholder lifetime `'_` in each affected return type:

- `Hive::root_key_node` returns `Result<KeyNode<'_, B>>`.
- `Hive::root_key_node_mut` returns `Result<KeyNodeMut<'_, B>>`.
- `KeyNode::class_name` returns `Option<Result<NtHiveNameString<'_>>>`.
- `KeyNode::name` returns `Result<NtHiveNameString<'_>>`.
- `KeyNodeMut::subkeys_mut` returns `Option<Result<SubKeyNodesMut<'_, B>>>`.
- `SubKeyNodesMut::next` returns `Option<Result<KeyNodeMut<'_, B>>>`.

The `'_` placeholder asks Rust to infer the same lifetime it inferred before, while making the lifetime-bearing type explicit. Method bodies, visibility, trait implementations, ownership, and runtime behavior remain unchanged. Do not introduce named method lifetimes or suppress the lint.

## Verification

Use the compiler warning as the regression test:

1. Preserve the recorded red result from `cargo rustc --lib --all-features --locked -- -D warnings` before source changes.
2. Run the same command after the six edits and require a successful, warning-free build.
3. Run `cargo fmt --check`, `git diff --check`, and `cargo test --all-features --locked`.

The full test suite must continue to report 15 passing unit tests and no failures.

## Scope

Change only the six return-type spellings in `src/hive.rs`, `src/key_node.rs`, and `src/subkeys_list.rs`. Do not add lint allowances, change crate-wide warning policy, alter dependencies, or perform unrelated lifetime refactoring.
