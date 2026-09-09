# PR #824 — Fresh-Eyes Final Review, CYCLE 3

**PR:** `S-25.02 cluster-2: artifact-sharding roll (BC-1.18.006 v1.11)`
**Head reviewed:** `68796531` (`feature/S-25.02-roll` → `develop`, merge-base `fff5e4cc`)
**Diff:** 9,933 additions / 2,122 deletions across 23 files

## VERDICT: REQUEST_CHANGES

The five claimed-fixed items from the prior cycle **all verify as genuinely fixed** — confirmed empirically, including by mutation-testing the code to prove the new tests are load-bearing rather than decorative. That work is solid.

However, this pass independently surfaced **three MAJOR findings** not addressed by the claimed remedies. One is a silent regression of this PR's *own* security fix caused by the composition of two fixes landed in different commits; two are in the `invoke.rs` / `main.rs` wiring, which the prior cycle's findings did not reach.

---

## Verification Performed

| Check | Command / method | Result |
|---|---|---|
| Full crate suite | `cargo test -p factory-dispatcher` | 454 lib + 24 integration + others, **0 failed** |
| Targeted shard tests | `cargo test -p factory-dispatcher --lib shard_manager::` | 161 passed, 0 failed |
| Formatting | `cargo fmt --check --all` | clean (exit 0) |
| Lint | `cargo clippy -p factory-dispatcher --all-targets -- -D warnings` | clean |
| **Mutation A** — reverted the `TempPathOccupied` typed distinction so it falls through to reclaim | rebuild + `cargo test test_N1` | `test_N1_..._temp_path_collision_never_misattributed` **FAILED** with exactly `SealedShardAlreadyExists` — the misattribution the fix exists to prevent |
| **Mutation B** — restored pre-`ef6ca3b4` unlink-then-stage ordering | rebuild + `cargo test test_N1` | `test_N1_..._failed_retry_staging_never_destroys_reclaimable_destination` **FAILED** with `NotFound` — destination destroyed |
| **Mutation C** — replaced ancestor walk with naive `parent().metadata()` | rebuild + `cargo test test_N2` | `test_N2_closest_existing_ancestor_propagates_when_blocked_by_a_file` **FAILED** |
| **Ordering probe** — moved `reclaim_identity_still_safe` to immediately before `remove_file` | full suite | **all pass** → proves no test pins the ordering, and proves the recommended fix is safe |
| Test seam scoping | grep for `test-support` gating, integration-test references | seam is `#[cfg(test)]`-only, not feature-gated, not reachable from `tests/` |
| Windows constant `0x0100` | `test_N4_..._rejects_symlink_via_o_nofollow` on darwin | passes → empirically proves `0x0100` is `O_NOFOLLOW` on **macOS**, contradicting the N-7 comment |
| Demo evidence | `ls` + `file` on `docs/demo-evidence/S-25.02/cluster-2-roll/` | 5 real GIF (1200×700) + 5 real WebM + 5 tapes + README — no `.txt` placeholders |
| Commit hygiene | `git log` scan | 56 commits, all Conventional, all story-scoped, **no `Co-Authored-By`**; `Claude-Session:` trailer matches established `develop` convention |
| CI | `gh pr checks 824` | **NOT green** — see gate note below |

Worktree was restored to pristine `68796531` after every mutation; `git status --porcelain` empty.

---

## Claimed-Fixed Items — Independent Confirmation

**1. N-1 call-site regression coverage for `publish_sealed_shard` — CONFIRMED FIXED.**
Both tests exist in `shard_manager::bc_1_18_006_roll_tests`, both call the real `pub fn publish_sealed_shard` (not `stage_temp_file_with_nonce`), and both are proven load-bearing by Mutations A and B. They assert on the *specific* behavior the fix changed — the error variant *and* its `E-SHD-001` Display text for misattribution, and post-condition on-disk state for destructive ordering — not merely "it doesn't crash".

**2. Windows `ERROR_PATH_NOT_FOUND` over-correction — CONFIRMED FIXED, better than the minimum asked for.**
The fix is not a `parent().metadata()` probe; it is `closest_existing_ancestor_is_directory_or_absent`, walking the full ancestor chain to the first extant node. That correctly handles the multi-level case a bare parent check would misjudge, which Mutation C confirms the suite pins. Genuine traversal-through-a-file still propagates (`test_FINDING1_..._false_for_path_traversal_through_non_directory`, `test_N2_..._propagates_when_blocked_by_a_file`). Compiling the helper under `#[cfg(any(windows, test))]` for portable unit-testing is a good call. See NIT-3 for the residual probe wrinkle.

**3. `FORCE_STAGE_FAILURE` test seam — CONFIRMED LEGITIMATE.**
`#[cfg(test)]` on the `thread_local!`, the enum, the guard, the setter, *and* the branch inside `stage_temp_file`. Not gated on `test-support` (which CI enables on the `build-dispatcher` matrix) — verified `test-support` remains `[]`, non-default, and touches only `VSDD_ASYNC_DRAIN_WINDOW_MS`. Not referenced from `tests/bc_1_18_006_roll_test.rs`. Pure early-return that fires only when a test arms it, with an RAII guard clearing state on drop including on unwind. Does not weaken the real path.

**4. Doc corrections (`O_NOFOLLOW` contradiction, FIFO `O_NONBLOCK` mechanism) — SUBSTANTIALLY FIXED.**
The N-5 correction is technically right and non-obvious: `O_RDONLY | O_NONBLOCK` on a writerless FIFO *succeeds* per POSIX (only `O_WRONLY` fails with `ENXIO`), and rejection genuinely comes from `is_file()` being false. The "must not remove `O_NONBLOCK` believing it is load-bearing for the open FAILING" note is good defensive documentation. See NIT-1 / NIT-2 for two statements not corrected.

**5. README + `publish_sealed_shard` doc vs. actual stage-before-unlink order — CONFIRMED ACCURATE.**
README line 46 and the `publish_sealed_shard` header both describe stage → unlink → publish, matching the code. (But see MAJOR-1: a *different* claim in those same paragraphs is now stale.)

**6. fmt / clippy / commit hygiene — CLEAN.**

---

## Findings

### MAJOR-1 — FIX-MED-1's TOCTOU re-check is no longer "immediately before the unlink"; Finding #2's fix silently widened the window it exists to close

`crates/factory-dispatcher/src/shard_manager.rs`, `publish_sealed_shard`

Actual execution order in the 0-byte-reclaim branch:

1. `reclaim_identity_still_safe(sealed_path)` — the FIX-MED-1 re-check
2. `stage_temp_file(sealed_path, content)` — **create + `write_all` + `sync_all`**
3. `std::fs::remove_file(sealed_path)` — the unlink
4. `publish_staged_temp_file(...)`

FIX-MED-1 (`0ea79c2c`) was introduced specifically so the identity re-check sits *adjacent* to the unlink. Finding #2's fix (`ef6ca3b4`) then inserted a full staged write — **including an `fsync`** — between them. The CWE-367 window FIX-MED-1 narrowed from "two adjacent syscalls" is now "one durable write wide". Concrete consequence: the exact data-loss scenario FIX-MED-1 was written to prevent — a legitimate concurrent writer replacing the 0-byte placeholder with real sealed content during step 2 has its content silently unlinked at step 3.

Four sites still assert the property that no longer holds:

- `shard_manager.rs:2989` — `// re-verify identity IMMEDIATELY BEFORE unlinking`
- `shard_manager.rs:3065-3068` — `reclaim_identity_still_safe`'s doc: *"called by `publish_sealed_shard`'s 0-byte reclaim path immediately before the unlink… closing the window as tightly as `std::fs` allows"*
- `shard_manager.rs:8994` — the Finding #8 test doc: *"which calls `reclaim_identity_still_safe` immediately before its `remove_file` unlink"*
- `docs/demo-evidence/S-25.02/cluster-2-roll/README.md:46` — *"IMMEDIATELY BEFORE unlinking"*

The PR body likewise still says FIX-MED-1 *"clos[es] most of the window"*.

No test catches this. The Finding #8 FIFO test gates on the re-check *existing*, not on where it sits, so it passes under either ordering. Moving the re-check to immediately before `remove_file` and running the full suite yields **all 454+ tests passing** — confirming both that nothing pins the current (worse) order and that the fix is safe.

**Suggested fix** — preserves *both* fixes fully (staging still precedes the unlink; the re-check becomes genuinely adjacent to it):

```rust
    let staged_tmp_path = match stage_temp_file(sealed_path, content) { /* ... unchanged ... */ };

    // FIX-MED-1: re-verify identity IMMEDIATELY before the unlink — after
    // staging, so no durable write widens the window between the two.
    if !reclaim_identity_still_safe(sealed_path) {
        let _ = std::fs::remove_file(&staged_tmp_path);
        return Err(already_exists_err());
    }

    if std::fs::remove_file(sealed_path).is_err() { /* ... unchanged ... */ }
```

Note the added `remove_file(&staged_tmp_path)` cleanup on the new early-return — without it the staged temp file leaks, the same leak class `stage_temp_file` already guards internally. Also add a test pinning the ordering (e.g. arm `FORCE_STAGE_FAILURE` for call #1 against a FIFO destination and assert the re-check's abort, not the staging failure, is what surfaces), and correct all four doc sites.

### MAJOR-2 — New catch-point-(i) leg reaches the destructive `execute_roll` without `validate_entry`, bypassing EC-022

`crates/factory-dispatcher/src/invoke.rs:2413-2431` (new in this PR)

`validate_entry` has exactly **one** production call site in the crate: `shard_manager.rs:1634`, inside `shard_cap_gate_check` (every other hit is a test or comment). The new path — `reconcile_replace_all_overcap_if_qualifying` → `find_matching_entry` → `reconcile_post_write_replace_all_overcap` → `execute_roll(entry, canonical_path, true)` at `shard_manager.rs:3899` — never validates the matched entry. `find_matching_entry` only compares `artifact_stem` against `file_stem()` and calls `path_falls_under_or_equals`; it validates nothing.

This matters because `execute_roll` is *destructive*: it seals and then truncates the canonical to 0 bytes. An entry with `artifact_path = "."` or `"./"` — which `validate_entry` rejects under EC-022, with dedicated tests — degenerates to an empty registered-component vector in `path_falls_under_or_equals`, whose suffix test then matches **any** path whose `file_stem()` equals `artifact_stem`. `shard_manager.rs:~4569` says so in its own words: *"This is precisely the raw behavior EC-022's `validate_entry` check exists to make unreachable via config."* On the PreToolUse leg such an entry is a loud `HookResult::Error`; on this new PostToolUse leg it silently seals-and-empties a file the config never legitimately governed. The same bypass covers the EC-010/011/013/015/017 cap-sanity checks — `reconcile_post_write_replace_all_overcap` compares `actual_size <= entry.shard_cap_bytes` using a cap the gate itself would refuse to operate under.

**Suggested fix:** call `validate_entry(&entry)` in `reconcile_replace_all_overcap_if_qualifying` immediately after `find_matching_entry` returns; on `Err`, emit a `tracing::warn!` and return `None` — consistent with this leg's documented fail-open-but-never-silent contract (Decision 15 point 2) and with the F-C2-P1-005 warn 25 lines above.

### MAJOR-3 — `main.rs` placement rationale asserts a PreToolUse guarantee that does not exist; both cited precedents are also wrong

`crates/factory-dispatcher/src/main.rs:291-311`

The new 30-line comment justifies placing the catch-point-(i) call before the `sync_tiers.is_empty() && partition.async_group.is_empty()` early return by invoking *"the exact 'silently stop firing if the plugin set changes' failure mode Decision 1's placement rule already exists to prevent **for the PreToolUse leg**."*

Traced: `shard_cap_precheck` is called at `executor.rs:532`, **inside** `execute_tiers`; `execute_tiers` is invoked at `main.rs:501` — *after* the early return at `main.rs:322`. So the PreToolUse byte-cap gate does exactly what the comment claims is already prevented: it silently stops firing on any configuration where no plugin matches. Both cited precedents fail the same way — `inject_git_context_if_qualifying` is at `main.rs:420` (behind the guard), and `write_indeterminate_marker` lives inside `execute_tiers` (per-plugin). The new call is in fact the *only* occupant of that slot.

Not merely a stale comment: it tells a future maintainer a safety property holds when it does not, and obscures a real asymmetry between the two legs of the same BC. Per CLAUDE.md's own rule (doc comment claiming a capability with no capability check → either implement the gate or remove the docs), either hoist the PreToolUse gate to match, or correct the comment (and its twin at `invoke.rs:2348-2352`) to state the asymmetry honestly and route the hoist decision to the architect.

### MINOR-1 — `find_matching_entry`'s `Err(DuplicateArtifactStem)` silently swallowed

`invoke.rs:2413-2416`: `find_matching_entry(&registry, &target_path).ok().flatten()?`. That variant is documented as a normal, fail-loud return callers MUST handle ("NOT a `panic!`-only-on-bug escape hatch"), and the PreToolUse path converts it to `HookResult::Error`. Here an ambiguous `[[shard]]` config produces zero telemetry — directly inconsistent with the F-C2-P1-005 fix 25 lines above, which added a `tracing::warn!` to the registry-load failure with the explicit rationale *"fail-open, never propagated as an error to the caller — but never silent."* Add the matching warn.

### MINOR-2 — Missing / non-string `tool_input.file_path` silently yields `None`

`invoke.rs:2408-2412`. The sibling PreToolUse path (`executor.rs:398-411`) treats the identical condition as fail-loud, with the stated rationale *"MUST fail loud, never silently resolve to an empty `PathBuf`."* Same "never silent" argument; a warn is the minimum.

### MINOR-3 — `main.rs:321` "UNCONDITIONAL native call, independent of the registry" is false

Two registry paths return before it: `resolve_registry_path()?` at `main.rs:152`, and the fail-open `return Ok(exit_code)` on `Registry::load` failure (~`main.rs:238`). A broken or unparseable `hooks-registry.toml` therefore silently disables catch point (i) — an over-cap canonical left unreconciled with no telemetry — while the doc asserts registry independence.

### MINOR-4 — "exercised end-to-end by `bc_1_18_006_roll_test.rs`'s AC-024 integration test" is not true of this call site

`main.rs:319-320`. That test calls `reconcile_replace_all_overcap_if_qualifying(&payload, dir.path())` directly (`tests/bc_1_18_006_roll_test.rs:603`, 1365, 1550); it never goes through `main::run`. No bats coverage either. The wiring itself — the placement, and the fact that the *canonicalized* `project_cwd` rather than raw `$CLAUDE_PROJECT_DIR` is joined with `.factory/shard-config.toml` — is untested. Either add coverage or correct the claim.

### MINOR-5 — Ordering side effect on PostToolUse plugins not considered

Because the catch-point-(i) call precedes the tier loop, PostToolUse WASM plugins registered for `Edit`/`MultiEdit` now execute against a canonical that may already have been truncated to 0 bytes. A PostToolUse validator that reads the artifact it was invoked for would see an empty file instead of the content just written — a plausible false-positive advisory/block source on precisely the dispatch where the roll fires. The 30-line placement rationale does not mention this trade-off; it should, and the risk should be assessed.

### MINOR-6 — New dependency's justification comment is factually wrong

`crates/factory-dispatcher/Cargo.toml:59-66` claims steps (b)-(d) *"all reuse `write_atomic`'s existing temp-file-then-rename primitive verbatim… no new atomic-write primitive is introduced."* Steps (c) and (d) do. Step (b) does **not**: `publish_sealed_shard` uses a purpose-built `write_exclusive` / `stage_temp_file` + `publish_staged_temp_file` pair over `OpenOptions::create_new` + `hard_link` + a local `random_nonce()` — deliberately *not* `rename(2)`, because rename clobbers and this path must be write-once. The dependency is still justified by (c)/(d); the stated scope is not. (The dependency itself is fine: workspace-internal path dep, `Cargo.lock` gains exactly one line, no new external crates, genuinely used in non-test code.)

### MINOR-7 — PR description is stale against the head being reviewed

The body twice cites `27d2f5ec` as "current head" and cites `39369cc6` for the "3072 tests / 0 failed" evidence; actual head is `68796531`, eight commits later. The cycle-2 tracking pointer is also hedged (*"…or a sibling `pr-review-cycle2.md`"*). Refresh before merge.

### NIT-1 — The N-7 constants comment is inverted and self-contradictory

`shard_manager.rs:3136-3137`: *"(`0x0100` is `O_NOFOLLOW` on Linux/Android but `O_NOCTTY` on Linux — the wrong flag entirely)"*. Backwards *and* internally contradictory (names Linux on both sides). The truth is the reverse: `0x0100` is `O_NOFOLLOW` on **macOS/BSD**, and `O_NOCTTY` on **Linux/Android** (Linux `O_NOFOLLOW` is `0o400000`). The **code is correct** — proven empirically: `test_N4_reclaim_identity_still_safe_rejects_symlink_via_o_nofollow` passes on a macOS host, which it could only do if `0x0100` really is `O_NOFOLLOW` on darwin. Flagged specifically because this comment is the entire payload of commit `e4c5d332`, whose stated purpose was getting these constants right — the classic "reworded, still wrong" shape.

### NIT-2 — `reclaim_identity_still_safe` doc overstates what it accepts

`shard_manager.rs:3094-3095`: *"Returns `false`… for a symlink whose target is non-empty."* On Unix, `O_NOFOLLOW` rejects **every** symlink regardless of target — as `test_N4` (which uses a symlink to a genuinely 0-byte target) demonstrates. The phrasing implies a symlink-to-empty-file would be accepted, true only of the `#[cfg(not(unix))]` fallback. Suggest: *"on Unix, for any symlink at all (`O_NOFOLLOW`); on other platforms, for a symlink whose target is non-empty."*

### NIT-3 — Ancestor probe conflates "absent" with "unstattable"

`closest_existing_ancestor_is_directory_or_absent`'s `Err(_) => cur = candidate.parent()` treats a permission or I/O failure identically to non-existence, walking further up until it finds a directory and relieving. Narrow in practice — an `EACCES` ancestor surfaces as `ERROR_ACCESS_DENIED` (5), which never reaches this helper via `is_genuinely_missing`'s `_ => false` arm — so no behavior change requested, only that the doc say the probe treats any inconclusive `metadata` as "absent".

### NIT-4 — `invoke.rs:2352-2354` reintroduces a retracted framing, and orders the discriminators backwards

*"Every check here is cheap and structural"* — `ShardRegistry::load` is a filesystem read plus a full TOML parse per qualifying dispatch. `find_matching_entry`'s own doc already retracts exactly this framing for the PreToolUse leg ("PR #818 fix-burst finding B4"). Relatedly, `shard_config_path.exists()` + `ShardRegistry::load` both run *before* `file_path` is extracted; extracting `file_path` first is free and skips the parse on a malformed payload.

### NIT-5 — Dependency direction

`last-amended-migrate` is a `clap`-based operator CLI crate; taking it as a normal (non-dev) dependency of the shipped dispatcher pulls `clap` into the dispatcher's build graph for one ~40-line `write_atomic`. Lock/CI impact is nil (`clap` is already a workspace dep, dead code is linker-dropped), but hoisting `atomic_write` into a leaf crate both could depend on would be the cleaner edge.

---

## Gate note — CI is not green at `68796531`

At review time: `validate`, `semgrep`, `deny-advisories`, `platforms-drift`, `policy-15`, `attestation-gate`, and `build-dispatcher (linux-arm64)` pass; **`cargo-host` (both runners), `bats-full-suite`, `bats-wave-handoff`, and `build-dispatcher` (darwin-arm64, darwin-x64, linux-x64, windows-x64) are all still pending.**

Specifically: `build-dispatcher (windows-x64)` is the **only** job in the matrix that executes the `#[cfg(windows)]` tests — including `test_N2_is_genuinely_missing_windows_path_not_found_relieved_for_missing_parent_dir`, this cycle's headline fix. `cargo-host` runs only ubuntu + macos. So the Windows arm of `is_genuinely_missing` has **zero executed coverage until that job goes green**. It must be green before merge, not merely "expected to pass".

---

## Checklist Summary

| # | Item | Result |
|---|---|---|
| 1 | Diff coherence | PASS — all changes scoped to `factory-dispatcher` + this story's demo evidence |
| 2 | Description accuracy | MINOR-7 (stale head SHAs) |
| 3 | Test coverage | Strong on `shard_manager.rs` (mutation-verified); gaps at MAJOR-1 (ordering), MINOR-4 (`main.rs` wiring) |
| 4 | Demo evidence | PASS — 5 real GIF+WebM, 1+ per AC, success and error paths, README accurate |
| 5 | Commit quality | PASS — Conventional, story-scoped, no AI co-author attribution |
| 6 | Diff size | 9,933 lines, far over the 500 guideline; ~3:1 test-to-production on a data-integrity path — prior disposition not to split is reasonable and concurred |
| 7 | Missing changes | MAJOR-2 (`validate_entry` not wired into the new leg) |
| 8 | Dependency status | PASS — cluster-1 (PR #818) merged; no new external crates |
