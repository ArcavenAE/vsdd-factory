# PR #824 — Fresh-Eyes Delta Review (cycle-7)

- **PR:** #824 — S-25.02 cluster-2, BC-1.18.006
- **Delta reviewed:** `74bfbec8..1693c4e0` (two cycle-6 NIT fixes)
- **covered_sha:** `1693c4e00dd0b5d9f584ab38daf33268a6b5bb68` (full 40-lowercase-hex; current live PR HEAD)
- **Prior approval baseline:** `74bfbec8` (cycle-6 delta review, APPROVED)
- **File touched:** `crates/factory-dispatcher/src/shard_manager.rs` (+95 / −12, 1 file)
- **Verdict:** **APPROVE / READY**

---

## VERDICT

**APPROVE / READY — covered_sha=`1693c4e00dd0b5d9f584ab38daf33268a6b5bb68`**

No blocking findings. No new findings requiring change. The delta is a correct, safe, production-grade hardening plus an accurate doc-parity fix, with a genuinely load-bearing, deterministic test. Merge-eligible.

---

## What the delta contains

Two cycle-6 NIT fixes, both scoped to `shard_manager.rs`:

- **NIT-A (doc-only):** the test-region banner comment previously cited `#[cfg(any(windows, test))]` for `closest_existing_ancestor_is_directory_or_absent`; updated to state unconditional compilation (no `#[cfg]` gate).
- **NIT-B (behavior hardening + new test):** the ancestor-walk `match` arm `Err(_) => cur = candidate.parent()` is split into `Err(e) if e.kind()==NotFound => cur = candidate.parent()` (strip + continue) and `Err(_) => return false` (propagate non-NotFound ancestor errors). New `#[cfg(unix)]` test `test_NIT_B_closest_existing_ancestor_propagates_non_notfound_ancestor_error`.

---

## Verification against the 5 required checks

### (1) NIT-B behavior change — correct and safe; common case unaffected — VERIFIED

Return-value contract of `closest_existing_ancestor_is_directory_or_absent`: `true` = relieve (treat as legitimate not-yet-created path → caller returns `Ok(0)`/`Ok(None)`/`Ok(vec![])`/`Ok(false)`); `false` = do not relieve, propagate the error.

The relieve-on-genuinely-missing path is byte-identical after the change:
- Immediate parent exists as directory → `Ok(meta)` arm → `meta.is_dir()` → `true` (unchanged).
- Multi-level fresh tree (intervening ancestors absent) → each level yields `NotFound` → the new `Err(e) if e.kind()==NotFound` arm walks up exactly as the old blanket `Err(_)` arm did → eventually hits an existing dir (`true`) or exhausts to `None` (`true`) (unchanged).
- Empty-component / no-existing-ancestor exits → unchanged (`true`).

Only the rare case changes: a **non-NotFound** error during the ancestor walk (e.g. `PermissionDenied`/`EACCES` on a search-blocked ancestor dir). Previously this was conflated into "walk up" and could be silently relieved to `Ok`; now it returns `false` → the error propagates. This is strictly more conservative and matches research doc §5 step-2. It introduces **no regression** to the relieve path — that case was never a genuinely-missing path — and in fact closes a latent silent-swallow of a real permission failure. Upstream, `is_genuinely_missing`'s own `err.kind() != NotFound → return false` early return still guarantees the ORIGINAL failing op's non-NotFound errors never route here; this arm governs only an inconclusive `metadata()` during the walk itself. Correct.

### (2) New test genuinely load-bearing and non-flaky — VERIFIED

- **Load-bearing:** fixture path `blocked/inner/child.md` with `blocked` chmod `0o000`. Walk step 1 = `metadata(blocked/inner)` → `EACCES` (`PermissionDenied`) because `blocked` has no search bit. New code: `Err(_)` arm → `return false`. Old (pre-NIT-B) code: blanket `Err(_) => cur = candidate.parent()` → `metadata(blocked)` (stat of `blocked` itself needs search on the *tempdir*, which is `0o755`) → `Ok`, `is_dir` → `true`. So old=`true`, new=`false`; the test's `assert!(!...)` distinguishes them exactly. Not a tautology — it fails on the pre-fix code.
- **Non-flaky:** `0o000` → `EACCES` is deterministic on POSIX for a non-root user. A precondition probe asserts the fixture yields `PermissionDenied` specifically (not `NotFound`) and fails loud with a descriptive message if run under privileged CI that bypasses permission bits — so the test can never silently mis-pass. A `RestorePermsOnDrop` guard restores `0o755` before the tempdir's `Drop` runs, preventing a leaked `0o000` dir from failing `remove_dir_all` with the same `EACCES`. Correct hygiene.

### (3) NIT-A accurate — VERIFIED

Confirmed neither `is_genuinely_missing` nor `closest_existing_ancestor_is_directory_or_absent` carries any `#[cfg]` attribute; the surrounding doc (lines 178-179) already states unconditional every-platform compilation. The old banner's `#[cfg(any(windows, test))]` citation was stale; the new wording ("compiled on every platform (no `#[cfg]` gate at all)") is accurate.

### (4) No regression to cycle-6 ancestor-walk or the 7 call sites — VERIFIED

The cycle-6-approved rewrite is preserved: the `Ok`, empty-component, and no-ancestor-exists paths are untouched, and the `NotFound` walk-up is preserved verbatim (only split out of the blanket arm). All 7 production call sites (`is_genuinely_missing` guards at the read/list/canonical/sealed paths, relieving to `Ok(0)`/`Ok(None)`/`Ok(vec![])`/`Ok(false)`) see identical behavior in the common case and strictly-safer propagation in the rare permission-error-during-walk case (mapping to genuine I/O error → `E-SHD-001` as a genuine non-NotFound failure already does on Unix). No call-site signature changed; no sibling-sweep gap.

### (5) New findings — NONE (one non-blocking observation)

- **OBSERVATION (no change required):** the new test fails loud (rather than skipping) if executed as root/privileged, since the `expect_err` precondition would not observe `PermissionDenied`. This is an intentional fail-loud design, consistent with the already-established workspace precedent (`internal_log.rs::silently_swallows_errors_on_read_only_dir`) cited in the test's own doc comment, and CI here is known non-root. Consistency with precedent is preferable to a divergent skip; not a finding.

---

## CI-safety spot checks

- **`non_snake_case`:** the uppercase test name `test_NIT_B_...` is allowed workspace-wide (`non_snake_case = "allow"` in root `Cargo.toml`, explicitly for BC-tracing test naming; `factory-dispatcher` opts in via `[lints] workspace = true`). Will not trip `-D warnings`.
- **Scope/deps:** `io` is in scope in `mod tests` (`use super::*` + top-level `use std::io;`); `tempfile` is a workspace dev-dependency; `std::os::unix::fs::PermissionsExt` imported locally in the test. Match arms are idiomatic and exhaustive — no new clippy or fmt exposure.

---

## Checklist coverage (delta scope)

1. Diff coherence — PASS (both hunks trace to the two cycle-6 NITs; nothing unrelated).
2. Description accuracy — PASS (commit subject describes NIT-A banner parity + NIT-B non-NotFound propagation).
3. Test coverage — PASS (NIT-B adds a load-bearing deterministic test; NIT-A is doc-only, no test owed).
4. Demo evidence — N/A for a NIT-fix delta on an already-approved PR (internal dispatcher helper; no AC surface change).
5. Commit quality — PASS (conventional `fix(S-25.02):` with story + BC id).
6. Diff size — PASS (95/12, single file).
7. Missing changes — PASS (both cycle-6 NITs present and complete).
8. Dependency status — N/A (delta review; no upstream dep change).
