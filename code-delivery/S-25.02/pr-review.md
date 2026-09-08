# PR #824 — Fresh-Eyes Review (S-25.02 cluster-2 "roll", BC-1.18.006 v1.11)

**PR:** #824 — `S-25.02 cluster-2: artifact-sharding roll (BC-1.18.006 v1.11)`
**Branch:** `feature/S-25.02-roll` → `develop`
**Head reviewed:** `8d17ffc44f155763832d3afec142f8b0df0c7a44` (merge-base `fff5e4cc`)
**Verdict:** **REQUEST_CHANGES** — 1 BLOCKING (external), 1 MAJOR, 5 MINOR, 2 NIT

> Supersedes the cluster-1 (PR #818) review previously at this path; that review's full text is
> preserved in this file's git history on `factory-artifacts` (commit `97e3c872`), matching the
> per-story pr-review.md convention cluster-1 itself established. This review is for cluster-2
> (PR #824), a distinct PR under the same story S-25.02.

Reviewed against the actual current diff at the head SHA (`gh pr diff 824`, `git show <sha>:<path>`),
including the 6 post-PR security-fix commits (`e67eb7ad`..`8d17ffc4`) — not against the fix-burst
narrative or any prior review's claims. Finding #2 was reproduced empirically with throwaway probes
(added, run, reverted; working tree confirmed clean). Posted to GitHub as a COMMENTED review plus a
test-coverage addendum, because GitHub refuses `--request-changes` on the author's own PR ("Can not
request changes on your own pull request"); the REQUEST_CHANGES verdict is stated in the body and
needs a human/second account to convert into a formal blocking review if branch protection is to
enforce it.

**Scope reviewed:** 6 code/config files + 16 demo-evidence files. `crates/factory-dispatcher/`:
`shard_manager.rs` (+6,012/−2,222; ~1,693 new production lines), `invoke.rs` (+227), `main.rs`
(+86/−21), `executor.rs` (+23/−10), `Cargo.toml` (+10), new `tests/bc_1_18_006_roll_test.rs`
(+2,108), plus `Cargo.lock` (+1). Test-to-production ratio ≈ 3:1.

---

## Summary

| # | Severity | Category | Finding |
|---|---|---|---|
| 1 | BLOCKING | dependency / CI | `cargo-host` red on both runners — merge gate unmet (root cause outside this diff) |
| 2 | MAJOR | correctness | `write_exclusive` temp-path collision misreported as `E-SHD-009`; destroys a reclaimable 0-byte destination |
| 3 | MAJOR | correctness / coverage | E-SHD-010 symlink guard absent AND untested on the Edit (~L1678) and MultiEdit (~L1748) arms |
| 4 | MINOR | correctness | `reclaim_identity_still_safe`'s `File::open` can block indefinitely on a FIFO |
| 5 | MINOR | description | PR body cites an orphaned commit SHA (`6cccf3a3`) not in HEAD (real one is `0ea79c2c`) |
| 6 | MINOR | description | Demo README's 0-byte-reclaim prose is stale after SEC-001 + FIX-MED-1 |
| 7 | MINOR | missing | `E-SHD-010` ships without an `error-taxonomy.md` entry and without a story anchor for the deferral |
| 8 | MINOR | coverage | FIX-MED-1 TOCTOU re-check tested only at helper level, never through `publish_sealed_shard` |
| 9 | NIT | correctness | `next_seal_seq`'s `max + 1` can overflow `u32` |
| 10 | NIT | size | 8,847 / 2,253 diff, far over the 500-line guideline (mitigated by 3:1 test ratio) |

---

## 1. [BLOCKING] CI is red on both runners — the PR's own "CI green" gate is unmet

`gh pr checks 824` reports `cargo-host (ubuntu-latest)` and `cargo-host (macos-latest)` both **fail**,
same step (`cargo test (workspace, all targets)`), same two tests:

```
panicked at crates/hook-plugins/validate-state-structure/src/lib.rs:2557:9:
assertion `left == right` failed: real STATE.md banner claims 386 lines but actual count is 395
```

**Not caused by this PR's diff.** Verified: locally on this branch `cargo test --workspace
--all-targets` is 3091 passed / 0 failed (the two failing tests read the CI-mounted `.factory/`
worktree, absent in my checkout). `develop`'s CI has been red for the last 5 consecutive runs
(2026-09-05 → 2026-09-07), including at merge-base `fff5e4cc`. Root cause: `validate-state-structure`
asserts against the live `.factory/STATE.md` banner `wc -l`, which is stale (386 vs 395).

Flagged BLOCKING because the PR's Pre-Merge Checklist lists "CI green" and that gate is objectively
unmet — not because the author introduced it. Route: `state-manager` (STATE.md banner reconcile on
`factory-artifacts`). Worth escalating separately: `develop` has merged with a non-functional
workspace-test gate for 3+ days, and coupling PR CI to a mutable external branch's content makes the
signal non-hermetic.

## 2. [MAJOR] `write_exclusive`'s temp-path collision is indistinguishable from a real sealed-shard collision

**File:** `shard_manager.rs`, `write_exclusive` (~L2435) and `publish_sealed_shard` (~L2495).

`write_exclusive` creates its temp file at a fully deterministic path `.{basename}.tmp-{pid}` with
`create_new(true)` (O_EXCL). It returns a bare `io::Result<()>`, so `publish_sealed_shard` cannot
tell "the temp path was occupied" from "the sealed destination was occupied" — and it assumes the
latter, routing any `AlreadyExists` into the destination-collision / 0-byte-reclaim path.

Reproduced with two throwaway probes (reverted; tree clean):

- **Probe A** — stale/planted temp file, no sealed shard on disk → `E-SHD-009: ... this seq already
  has durable content on disk`, while `sealed exists = false`. Factually false, and self-perpetuating
  (nothing removes the temp file).
- **Probe B** — legitimately reclaimable 0-byte destination + stale temp file → the reclaim path
  lstat'd the destination, passed `reclaim_identity_still_safe`, **unlinked the destination**, retried
  `write_exclusive`, hit the same temp file, and failed. A failed op that nonetheless deleted a file —
  contradicting the function's own "leave it byte-identical and untouched" contract.

No attacker required: `write_exclusive`'s cleanup line sits after two `?` operators
(`write_all`/`sync_all`), so any `ENOSPC`/`EIO` leaks the temp file permanently. **Fix:** random
nonce in the temp path (crate already uses `tempfile` in tests) and/or a typed error distinguishing
temp-path-occupied from destination-exists; scope-guard the temp file so it is removed on every
early return.

## 3. [MAJOR] E-SHD-010 symlink guard absent AND untested on the Edit/MultiEdit arms

**File:** `shard_manager.rs`, `shard_cap_gate_check`.

The PR body says FIX-MED-2 was applied "at all 4 sites" — confirmed four `reject_canonical_symlink`
callsites (Write ~L1645, `execute_roll` ~L2731, `self_heal_resume_from_truncate` ~L3061,
`reconcile_post_write_replace_all_overcap` ~L3325). But the **Edit** arm (~L1678) and **MultiEdit**
arm (~L1748) call `current_shard_bytes_flat(target_path)` — which uses `std::fs::metadata` (follows
symlinks) — with no preceding guard. So two of three mutation-tool arms `stat()` through a symlinked
canonical while only Write is guarded. `read_changelog_item_count`'s `File::open` (~L1244) is likewise
unguarded and reads content.

This path has **zero test coverage**: `test_FIXMED2_..._e_shd_010` (~L7846) exercises only the
Write-arm backstop; there is no Edit or MultiEdit variant, so the asymmetry would not regress-fail.
This is the missed-sibling-callsite pattern TD-VSDD-060 exists to catch, shipped without a pinning
test — hence MAJOR rather than MINOR.

Impact is bounded (if the leaked size trips the trigger, `execute_roll`'s guard at ~L2731 refuses
loud, so no symlink-target bytes are ever sealed) — residual is info-exposure + guard asymmetry, not
data loss. **Route:** implementer adds `reject_canonical_symlink` at ~L1678/~L1748 (or hoist it above
the `match tool_kind` so all three arms inherit it); test-writer adds Edit + MultiEdit variants of
the ~L7846 test.

## 4. [MINOR] `reclaim_identity_still_safe` can block indefinitely on a FIFO

**File:** `shard_manager.rs` (~L2633). `std::fs::File::open(path)` on a FIFO with no writer blocks
forever, hanging the PreToolUse gate. Window is narrow (swap a FIFO between the `symlink_metadata`
probe and this call), but the consequence is a stalled dispatch. **Fix:** on Unix, open with
`O_NONBLOCK | O_NOFOLLOW` via `OpenOptionsExt::custom_flags` — also closes the residual symlink case
the doc comment currently documents as unclosable. No new dependency.

## 5. [MINOR] PR body cites a commit not in this PR

"Deferred to F6" cites `6cccf3a3` as "this PR's own FIX-MED-1 commit"; it is **not an ancestor of
HEAD** (`git merge-base --is-ancestor` → NO; parent `f874cd1e`). The in-HEAD FIX-MED-1 is `0ea79c2c`
(parent `0f56530d`), identical subject line — `6cccf3a3` is an orphaned artifact of a discarded
branch state. The Security Review table cites the correct SHA; the deferrals section should too.

## 6. [MINOR] Demo README describes pre-security-fix behavior

`docs/demo-evidence/S-25.02/cluster-2-roll/README.md`, EC-025 row, says the reclaim `"stat()`s
once"`. As of SEC-001 the probe is `lstat` (non-dereferencing) and FIX-MED-1 adds an open-handle
re-check before unlink. The recording is still valid (test unchanged, still passes); the prose
describes the pre-fix mechanism. One-line update.

## 7. [MINOR] `E-SHD-010` ships without a taxonomy entry or a deferral anchor

Correctly routed to product-owner per the Agent Routing Table, but recorded as "flagged for a
follow-up documentary pass" with **no story/wave anchor** — Canonical Principle Rule 3 requires a
named future story so the deferral cannot get lost. Attach a real story ID, or have product-owner add
the single table row in-scope.

## 8. [MINOR] FIX-MED-1 TOCTOU re-check tested only at helper level

Both tests (~L7973, ~L8008) call `reclaim_identity_still_safe` directly; no test drives the
probe → concurrent-write → re-check → unlink path through `publish_sealed_shard` itself. The
integration guarantee rests on inspection of the callsite (~L2564), not a test. This intersects
Finding #2 — an integration-level test through `publish_sealed_shard` would likely have surfaced the
temp-path-collision misattribution.

## 9. [NIT] `next_seal_seq` arithmetic can overflow

`...max().unwrap_or(0) + 1` on `seq: u32` panics in debug / wraps in release at `u32::MAX`.
Practically unreachable; `checked_add(1)` → fail-loud error is one line and matches the module's
discipline.

## 10. [NIT] Diff size

8,847 / 2,253 far over the 500-line guideline, but ≈3:1 test-to-production on a data-integrity path
is the right trade. Noted so size is not mistaken for scope creep.

---

## Verified clean (no rubber-stamp)

- **Diff coherence:** every changed file within `crates/factory-dispatcher/`,
  `docs/demo-evidence/S-25.02/`, or `Cargo.lock`. No unrelated changes.
- **Commit quality:** all 42 commits Conventional-Commits format, all carry the story ID, **no AI
  attribution** in any commit body.
- **Forbidden patterns:** no `println!`/`eprintln!`/`dbg!` in production code; no
  `unwrap`/`expect`/`panic!`/`todo!`/`unimplemented!` in the non-test half of `shard_manager.rs`
  (matches are inside comments only).
- **Dependencies:** single new dep `last-amended-migrate` is workspace-internal, path-pinned,
  justified inline vs ADR-051 §8 acyclicity. `Cargo.lock` gains one line, no new third-party packages.
- **`main.rs` refactor:** `resolve_project_cwd` is a faithful extract of the pre-existing
  `base_host_ctx.cwd` logic (same env var, empty-filter, canonicalize-with-fallback, `current_dir`
  fallback); `SEC-004 TOCTOU ACCEPTED` rationale preserved. No behavior change for other plugins.
- **Path resolution:** `invoke.rs` uses raw `file_path` for target + `cwd` join for config —
  consistent with `executor.rs` and the harness's absolute-`file_path` invariant. Not a bug.
- **Catch-point (i) disposition:** `reconcile_replace_all_overcap_if_qualifying` only
  `tracing::warn!`s on failure — correct for a PostToolUse janitor that cannot block; catch point (ii)
  fails loud on the next dispatch, so the condition is recoverable, not swallowed.
- **Demo evidence:** 5 `.gif` + 5 `.webm` + 5 `.tape` + README, ≥1 per AC (AC-006 ×3, AC-007 ×2),
  success and error paths both recorded, each `.tape` invokes a real `cargo test ... --exact
  --nocapture`, all five named tests exist. Genuine recordings, not `.txt` placeholders.
- **Security-fix coverage:** FIX-HIGH-1 → ~L7909, SEC-001 → ~L6645, SEC-002 → ~L3949/~L3982,
  SEC-003 → ~L3809/~L3827, FIX-MED-2's four guarded sites → ~L7846/~L8026/~L8070/~L8104. No
  `#[ignore]`, no `should_panic`, no tautological/over-mocked tests.
- **Dependency PR:** cluster-1 (BC-1.18.005, PR #818) merged to develop, as claimed.

---

## Verdict

**REQUEST_CHANGES.** Finding #2 (reproducible false `E-SHD-009` + destination deletion on a failed
op) and Finding #3 (unguarded, untested symlink `stat()` on Edit/MultiEdit) are the substantive
ones and should be fixed in-scope per the production-grade default. Finding #1 blocks merge
mechanically but belongs to `state-manager`, independent of this branch. #4–#8 are worth closing
in-scope; #9–#10 optional.
