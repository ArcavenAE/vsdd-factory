# PR #824 — Fresh-Eyes PR Review (CYCLE 2)

**PR:** #824 — `S-25.02 cluster-2: artifact-sharding roll (BC-1.18.006 v1.11)`
**Branch:** `feature/S-25.02-roll` → `develop`
**Head reviewed:** `27d2f5ec02781150c1fcd138bd791df4e0efdacc`
**Merge-base:** `fff5e4cc`
**Diff:** 23 files, +9639 / −2233
**Reviewer:** pr-reviewer (fresh context, diff-only; cycle-1 `pr-review.md` deliberately NOT read)

## VERDICT: REQUEST_CHANGES

One MAJOR gating finding (N-1). Everything else is MINOR/NIT.

---

## Disposition of the 5 claimed cycle-1 fixes

| # | Claimed fix | Commit | Disposition |
|---|---|---|---|
| 1 | Windows `read_canonical_content` NotFound disambiguation + sibling sweep | `c761c5af` | **RESOLVED** — but introduces a new Windows regression (N-2) |
| 2a | `write_exclusive` stage-then-publish (temp-collision misattribution + destructive reclaim) | `ef6ca3b4` | **RESOLVED IN CODE / NOT VERIFIED** — real structural fix, zero call-site regression coverage (N-1) |
| 2b | Symlink guard on Edit/MultiEdit arms | `ef6ca3b4` | **RESOLVED** — genuinely hoisted + 3 new tests, all green |
| 3 | FIFO-open hang + u32 seq overflow | `c3c87bd6` | **RESOLVED** — both real and tested; mechanism misdescribed in docs (N-5) |
| 4 | Stale demo README | `ac56f244` | **RESOLVED, THEN RE-STALED** by `ef6ca3b4` (N-3) |
| 5 | Real call-site test for TOCTOU re-check | `27d2f5ec` | **RESOLVED** — genuinely load-bearing |

## Verification performed

- Ran `cargo test -p factory-dispatcher --lib shard_manager::` → **155 passed, 0 failed**. All 16 `FINDING*` / `FIXMED*` / `FIXHIGH1` tests green.
- `reject_canonical_symlink` is genuinely hoisted above `match tool_kind` (`shard_manager.rs:1627`), covering Write/Edit/MultiEdit from one call, plus a second guard before the frontmatter arm's `read_changelog_item_count` (`:1977`). Three new arm-specific tests assert `E-SHD-010` and that the symlink target is untouched. Not a paper fix.
- `publish_sealed_shard` genuinely stages before unlinking (`:2859` stage → `:2876` unlink → `:2881` publish); `TempPathOccupied` maps to `SealWriteFailed`/E-SHD-001 while only `DestinationOccupied` enters reclaim. Destructive reclaim-then-fail is gone.
- `next_seal_seq` uses `checked_add` (`:2426`) with a real `u32::MAX` fixture test.
- `test_FIXMED1_publish_sealed_shard_callsite_...` drives reclaim through the real `publish_sealed_shard` entrypoint and asserts the FIFO is never unlinked — genuine integration-level TOCTOU coverage.
- Commit hygiene clean: Conventional Commits, story ID per subject, no `Co-Authored-By: Claude`, no robot emoji. `Claude-Session:` trailer matches established convention on `develop` (#816/#817/#818).
- Diff coherent: the 5 fix commits touch only `shard_manager.rs` + the demo README. `invoke.rs`/`main.rs`/`executor.rs` changes all predate the cycle-1 review. No unrelated changes.
- CI at review time: `SAST (Semgrep)`, `deny-advisories`, `validate`, `policy-15-attestation-location` **pass**; `cargo-host` (both runners), bats, platform jobs **pending**. No new failure attributable to this diff.

---

## Findings

### N-1 — MAJOR — Finding #2's fix has no call-site regression coverage (both halves)

**Category:** coverage
**File:** `crates/factory-dispatcher/src/shard_manager.rs` (`publish_sealed_shard`, `stage_temp_file`)

`ef6ca3b4` is a correct structural fix for a data-destruction defect, but nothing in the suite would catch its reversion. Every test-region reference to `stage_temp_file`, `TempPathOccupied`, `DestinationOccupied`, and `WriteExclusiveError` was grepped; the only hit is `test_FIXHIGH1_write_exclusive_refuses_to_follow_preplanted_symlink_at_temp_path`, which calls `stage_temp_file_with_nonce` **directly** and asserts `StageError::TempPathOccupied`. That covers the helper's error taxonomy, not the two behaviors Finding #2 was about:

1. **Misattribution** — no test drives `publish_sealed_shard` into a temp-path collision to assert `SealWriteFailed`/E-SHD-001 rather than the misleading `SealedShardAlreadyExists`/E-SHD-009. Reverting the `TempPathOccupied` arm to fall through into the reclaim block passes the whole suite.
2. **Destructive reclaim** — no test asserts that a staging failure on the reclaim retry leaves the reclaimable 0-byte destination on disk. Restoring the pre-fix unlink-then-stage ordering passes the whole suite.

This is the same gap class cycle-1 Finding #8 identified for FIX-MED-1, which the author closed correctly in `27d2f5ec` ("Both existing FIX-MED-1 tests call `reclaim_identity_still_safe` directly, so a regression in how `publish_sealed_shard` actually wires the re-check in would go undetected"). That reasoning applies verbatim to Finding #2's own fix and was not applied. Under TD-VSDD-059, a MAJOR fix with no load-bearing assertion is not done.

**Suggestion:** add two deterministic call-site tests, no new deps, pinning the nonce only in the test path exactly as `test_FIXHIGH1_*` already does:
- pre-plant a non-empty destination **and** the exact temp path → assert `Err(SealWriteFailed)` (E-SHD-001), never E-SHD-009;
- 0-byte destination + occupied temp path (a directory at the temp path forces `StageError` deterministically) → assert `Err(SealWriteFailed)` **and** `assert!(sealed_path.exists())`. That last assertion is what actually pins the stage-before-unlink ordering.

### N-2 — MINOR — `is_genuinely_missing` over-corrects on Windows: a missing parent directory now fails loud

**Category:** coherence / regression
**File:** `crates/factory-dispatcher/src/shard_manager.rs:118`

On Windows, `ERROR_PATH_NOT_FOUND` (3) covers two distinct situations: a path traversing through a non-directory (the genuine error targeted) **and** a legitimately not-yet-created intermediate directory. The fix propagates both.

Windows-only consequence: a first `Write` to a governed artifact whose parent directory does not exist yet (e.g. the first `burst-log.md` in a freshly-bootstrapped cycle directory) previously hit `current_shard_bytes_flat` → `Ok(0)` → `Continue`. It now returns `Err` → `BackstopProbeFailed` → `HookResult::Error`, blocking the write. The gate is `PreToolUse`, so the directory genuinely does not exist at hook time.

Unix is unaffected (missing intermediate dir is plain `ENOENT`/`NotFound`, still relieved; traversal-through-a-file is the distinct `NotADirectory` kind). `test_FINDING1_is_genuinely_missing_windows_path_not_found_propagates` encodes the over-correction rather than catching it, and no Windows runner executes these tests.

**Suggestion:** on the Windows arm, relieve `ERROR_PATH_NOT_FOUND` when the parent directory is absent and propagate only when the parent exists but is not a directory — the case Unix distinguishes as `NotADirectory`. A `parent().metadata()` probe restores parity in both directions.

### N-3 — MINOR — README and `publish_sealed_shard` doc comment re-staled by `ef6ca3b4`

**Category:** description / docs
**Files:** `docs/demo-evidence/S-25.02/cluster-2-roll/README.md`; `crates/factory-dispatcher/src/shard_manager.rs:2756`

`ac56f244` fixed the README's 0-byte-reclaim prose; `ef6ca3b4` then changed the mechanism without updating either description. Both now misdescribe the shipped order:

- README AC-006 0-byte-reclaim row: "only then does it unlink the 0-byte file and retry `write_exclusive` exactly ONCE". The code stages **first**, then unlinks, then calls `publish_staged_temp_file`; `write_exclusive` is not called on the retry path at all.
- `publish_sealed_shard` doc comment: "it `unlink`s it and retries [`write_exclusive`] EXACTLY ONCE" — same two inaccuracies.

The stage-before-unlink guarantee is the point of the fix; both places still document the defect. This branch already carries a `docs(S-25.02): F-C2-P1-003 — sweep stale ... claims (TD-VSDD-060)` commit, so the discipline is established.

### N-4 — MINOR — `reclaim_identity_still_safe` doc comment self-contradicts; new O_NOFOLLOW property untested

**Category:** docs / coverage
**File:** `crates/factory-dispatcher/src/shard_manager.rs:2912–2950`

Lines 2912–2926 still assert "no `O_NOFOLLOW` primitive is available from `std::fs` alone, and adding one would require a new dependency"; lines 2936–2950 state the opposite — O_NOFOLLOW **is** now passed via `custom_flags`, with no new dependency. `c3c87bd6` appended a correcting paragraph instead of rewriting the superseded one, so the function's contract reads two ways. A reader stopping at the first paragraph concludes the symlink gap is open.

Separately, the newly-claimed security property (symlink at `path` → open fails → `false` → reclaim aborts) has no test; the two existing FIX-MED-1 helper tests cover only the content-changed and missing-path arms.

**Suggestion:** rewrite the superseded paragraph rather than layering a contradiction, and add a one-line `symlink()` fixture asserting `!reclaim_identity_still_safe(&path)`.

### N-5 — NIT — Finding #4's stated mechanism is wrong (the fix works anyway)

**Category:** docs
**File:** `crates/factory-dispatcher/src/shard_manager.rs:2936` + commit `c3c87bd6` message

Both claim O_NONBLOCK means "a FIFO fails the open immediately instead of waiting for a writer that will never connect." Per POSIX, `O_RDONLY | O_NONBLOCK` on a FIFO **succeeds** immediately; it is `O_WRONLY | O_NONBLOCK` with no reader that fails (`ENXIO`). The hang is genuinely fixed, but by a different mechanism: the open returns a valid fd, then `metadata().file_type().is_file()` is `false` for a FIFO. The tests encode the correct behavior; only the prose is wrong. Worth correcting so a future reader does not remove the flag believing it is load-bearing for the open failing.

### N-6 — NIT — PR description cites a SHA that is not in the branch

**Category:** description
The "Deferred to F6" section credits FIX-MED-1 to `6cccf3a3`, which exists in the object store but is **not an ancestor of HEAD** (rewritten); the in-branch commit is `0ea79c2c`. The Test Evidence table's "3072 tests / 0 failed (last code commit, `39369cc6`)" is five code commits stale.

### N-7 — NIT — hardcoded `O_NONBLOCK`/`O_NOFOLLOW` wrong for non-Linux, non-BSD unix

**Category:** coherence
**File:** `crates/factory-dispatcher/src/shard_manager.rs:2956–2964`
The cfg splits on `target_os = "linux"` vs everything-else-unix. Android is Linux-ABI but not `target_os = "linux"`, so it would receive BSD values (`0x0100` = `O_NOCTTY` on Linux) and the O_NOFOLLOW guard would quietly not apply. Not a shipped target (release builds are darwin-arm64/x86_64, linux-x86_64, linux-musl, windows-x86_64; musl is `target_os = "linux"`), so this is theoretical. `#[cfg(any(target_os = "linux", target_os = "android"))]` fixes it for free.

---

## Path to APPROVE

**N-1 alone gates.** Add the two `publish_sealed_shard` call-site tests described above — the same remedy `27d2f5ec` already applied to FIX-MED-1, applied to the other MAJOR fix in the same burst. N-3 and N-4's doc corrections are ~10 minutes and should fold into the same commit under this repo's stale-doc discipline. N-2 is a genuine Windows regression but contained and untriggerable on the Unix runners. N-5/N-6/N-7 are cosmetic.

## Explicitly out of scope

The residual LOW-severity CWE-367 TOCTOU sub-window in `reclaim_identity_still_safe` is acknowledged as present but excluded from this verdict per human direction — it is being adjudicated separately and is not a gating finding here.
