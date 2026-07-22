# Lifetime-Elision Warning Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate all six `mismatched_lifetime_syntaxes` warnings without changing inferred lifetimes or runtime behavior.

**Architecture:** Make the existing inferred lifetimes visible by adding Rust's anonymous `'_` lifetime to six generic return types. Use the compiler with warnings denied as the red/green regression test, then run the complete unit-test suite.

**Tech Stack:** Rust 2021, Cargo, rustc 1.95 diagnostics; crate MSRV remains Rust 1.81.

## Global Constraints

- Work on branch `fix/lifetime-elision-warnings` in the current checkout; do not create a worktree.
- Change only six return-type spellings in `src/hive.rs`, `src/key_node.rs`, and `src/subkeys_list.rs`.
- Use anonymous `'_` lifetimes; do not add named method lifetimes or lint allowances.
- Keep method bodies, visibility, ownership, runtime behavior, dependencies, and `Cargo.lock` unchanged.
- The baseline has 15 passing unit tests and exactly six `mismatched_lifetime_syntaxes` warnings.

---

### Task 1: Make inferred return lifetimes explicit

**Files:**

- Modify: `src/hive.rs:213`
- Modify: `src/hive.rs:387`
- Modify: `src/key_node.rs:467`
- Modify: `src/key_node.rs:472`
- Modify: `src/key_node.rs:576`
- Modify: `src/subkeys_list.rs:245`
- Test: compiler warning gate and existing inline unit tests

**Interfaces:**

- Consumes: the existing elided borrow relationship inferred from `&self` or `&mut self`.
- Produces: the same six methods with `'_` written in their lifetime-bearing return types; no new interface.

- [ ] **Step 1: Re-run the warning gate and verify the red failure**

Run:

```bash
cargo rustc --lib --all-features --locked -- -D warnings
```

Expected: FAIL with exit code 101 and exactly six `mismatched_lifetime_syntaxes` diagnostics at `src/hive.rs:213`, `src/hive.rs:387`, `src/key_node.rs:467`, `src/key_node.rs:472`, `src/key_node.rs:576`, and `src/subkeys_list.rs:245`.

- [ ] **Step 2: Apply the six minimal signature changes**

Apply this exact diff:

```diff
--- a/src/hive.rs
+++ b/src/hive.rs
@@
-    pub fn root_key_node(&self) -> Result<KeyNode<B>> {
+    pub fn root_key_node(&self) -> Result<KeyNode<'_, B>> {
@@
-    pub(crate) fn root_key_node_mut(&mut self) -> Result<KeyNodeMut<B>> {
+    pub(crate) fn root_key_node_mut(&mut self) -> Result<KeyNodeMut<'_, B>> {
--- a/src/key_node.rs
+++ b/src/key_node.rs
@@
-    pub fn class_name(&self) -> Option<Result<NtHiveNameString>> {
+    pub fn class_name(&self) -> Option<Result<NtHiveNameString<'_>>> {
@@
-    pub fn name(&self) -> Result<NtHiveNameString> {
+    pub fn name(&self) -> Result<NtHiveNameString<'_>> {
@@
-    pub(crate) fn subkeys_mut(&mut self) -> Option<Result<SubKeyNodesMut<B>>> {
+    pub(crate) fn subkeys_mut(&mut self) -> Option<Result<SubKeyNodesMut<'_, B>>> {
--- a/src/subkeys_list.rs
+++ b/src/subkeys_list.rs
@@
-    pub fn next(&mut self) -> Option<Result<KeyNodeMut<B>>> {
+    pub fn next(&mut self) -> Option<Result<KeyNodeMut<'_, B>>> {
```

- [ ] **Step 3: Re-run the warning gate and verify green**

Run:

```bash
cargo rustc --lib --all-features --locked -- -D warnings
```

Expected: PASS with exit code 0 and no warnings.

- [ ] **Step 4: Verify formatting, diff hygiene, and behavior**

Run:

```bash
cargo fmt --check
git diff --check
cargo test --all-features --locked
```

Expected: formatting and diff checks produce no output; all 15 unit tests and all doctests pass with no compiler warnings.

- [ ] **Step 5: Confirm the change is limited to six signature lines**

Run:

```bash
git diff --stat
git diff -- src/hive.rs src/key_node.rs src/subkeys_list.rs
```

Expected: three source files changed with six insertions and six deletions; every changed line adds only `'_` to a return type.

- [ ] **Step 6: Commit the implementation**

```bash
git add src/hive.rs src/key_node.rs src/subkeys_list.rs
git commit -m "fix: clarify elided return lifetimes"
```
