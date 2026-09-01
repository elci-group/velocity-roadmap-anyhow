# Detailed Roadmap: Replacing `anyhow` in Velocity

## 1. Current usage audit

`anyhow` appears in the following crates/files:

- `crates/velocity-cli/src/main.rs`
- `crates/velocity-cli/src/hud_config.rs`
- `crates/velocity-diagnostics/src/audit.rs`
- `crates/velocity-diagnostics/src/replay.rs`
- `crates/velocity-elci/src/deckhand.rs`
- `crates/velocity-elci/src/tempcheq.rs`
- `crates/velocity-elci/src/uni.rs`
- `crates/velocity-plugin-runtime/src/lib.rs`
- `crates/velocity-pty/src/lib.rs`
- `crates/velocity-test/src/lib.rs`
- `crates/velocity-test/tests/conformance.rs`

The two patterns in use are:

1. `anyhow::Result<T>` as a return type.
2. `anyhow!("context: {e}")` to wrap a lower-level error.

`thiserror` is already a workspace dependency, so the project has already bought into typed errors in the plugin crate.

## 2. Replacement strategy

Adopt a **library-first, typed-error strategy**:

- **Library crates** (`velocity-core`, `velocity-events`, `velocity-elci`, `velocity-plugin-runtime`, `velocity-pty`, `velocity-diagnostics`, `velocity-test`) replace `anyhow::Result<T>` with `Result<T, CrateError>` where `CrateError` is a `#[derive(Debug, thiserror::Error)]` enum.
- **Application / CLI crate** (`velocity-cli`) may keep a small, project-local `error.rs` that aggregates library errors into a top-level `cli::Error`. The CLI can still use a single error type for `main()`, but it should not depend on `anyhow`.
- Replace `anyhow!("...")` calls with typed variants such as `CrateError::TempcheqExec { source }` or `CrateError::Io { context, source }`.

## 3. Step-by-step migration

1. **Inventory** — list every `anyhow::Result` and `anyhow!` call site.
2. **Design error enums** — per crate, define an error enum covering the existing error contexts. Re-use `std::io::Error` and other standard error types via `#[from]` where appropriate.
3. **Migrate one crate at a time**, starting with the leaf crates (`velocity-pty`, `velocity-diagnostics`) and moving inward to `velocity-cli`.
4. **Convert call sites**:
   - `anyhow::Result<T>` → `Result<T, crate::Error>`
   - `anyhow!("failed to execute tempcheq: {e}")` → `Error::TempcheqExec { source: e }`
   - `.context("...")?` (if used) → explicit context variants.
5. **Add `From` impls** so `?` continues to work for `std::io::Error`, `serde_json::Error`, etc.
6. **Update tests** that rely on `anyhow::Result`.
7. **Remove `anyhow` from workspace dependencies** once all call sites are gone.
8. **Run `cargo test`, `cargo clippy`, and the conformance suite**.

## 4. Risk and effort assessment

| Risk | Level | Mitigation |
|---|---|---|
| Error message churn | Medium | Keep string formatting identical in typed variants; add unit tests for `Display`. |
| `?` ergonomics loss | Low | Use `thiserror#[from]` for std and serde errors. |
| Dyn/CLI aggregation | Medium | Define a small top-level CLI error enum. |
| API breakage | High | Many public functions return `anyhow::Result`; this is a breaking change for any external callers. Coordinate with versioning. |

**Estimated effort:** 2–3 days of focused refactoring.

## 5. Acceptance criteria

- [ ] `anyhow` no longer appears in `Cargo.lock` for the workspace.
- [ ] `cargo test` passes.
- [ ] `cargo clippy` passes with no new warnings.
- [ ] No remaining `anyhow::Result`, `anyhow!`, or `Context` imports.
- [ ] Error messages remain human-readable (snapshot tests if available).
- [ ] Conformance tests pass.

## 6. Test plan

1. Before migration, capture representative error messages as snapshots or unit-test asserts.
2. After migration, assert the same messages are produced.
3. Introduce an artificial I/O failure in each migrated crate and verify the error variant is correct.
