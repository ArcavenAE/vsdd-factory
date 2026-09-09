# PR #824 — Fresh-Eyes Final Review, CYCLE 4

**PR:** `S-25.02 cluster-2: artifact-sharding roll (BC-1.18.006 v1.11)`
**Head reviewed:** `27bbc6f8` (`feature/S-25.02-roll` → `develop`, merge-base `fff5e4cc`) — confirmed current via `gh pr view 824 --json headRefOid`, no discrepancy.
**Diff:** 11,100 additions / 2,319 deletions across 31 files (all scoped to `crates/factory-dispatcher/` + this story's demo evidence).

## VERDICT: APPROVE

All three cycle-3 MAJOR findings verify as **genuine, load-bearing fixes** — confirmed empirically, including by mutation-testing MAJOR-1 and MAJOR-2 (reverting each claimed fix makes its pinning test fail with exactly the defect the finding described). The 12 cycle-3 MINOR/NIT items are addressed in code with two exceptions noted below. Zero MAJOR/blocking issues remain.

Two non-blocking **MINOR** items are open: one **new** sweep finding introduced by the MAJOR-3 hoist (a fired PreToolUse shard-gate verdict, whose destructive roll has already run, can be silently downgraded to exit 0 on the rare `build_engine()`-failure path), and the still-stale PR body (cycle-3 MINOR-7, never touched by any remediation commit). Neither gates code correctness.

**Merge precondition (not a review finding):** CI is **not green** — `build-dispatcher (windows-x64)` (the only job that executes the `#[cfg(windows)]` `is_genuinely_missing` tests, this PR's N-2 fix) is still pending, along with the rest of the build-dispatcher matrix, both `cargo-host` runners, and `bats-full-suite`. These must land green before merge exactly as cycle-3 required.

---

## Verification Performed

| Check | Command / method | Result |
|---|---|---|
| Formatting | `cargo fmt --check --all` | clean (exit 0) |
| Lint | `cargo clippy -p factory-dispatcher --all-targets -- -D warnings` | clean (exit 0) |
| Shard-manager lib tests | `cargo test -p factory-dispatcher --lib shard_manager::` | **162 passed, 0 failed** |
| Cluster-2 + cluster-1 integration | `cargo test --test bc_1_18_006_roll_test --test bc_1_18_005_shard_cap_trigger_test` | **0 failed** (incl. `test_MAJOR2_*`, `test_MINOR4_*`) |
| MAJOR-3 real-binary falsifier | `test_MAJOR3_shard_cap_gate_fires_via_real_binary_when_no_plugin_matched` | **pass** — exit 2 with empty `hooks-registry.toml` |
| **Mutation A** (MAJOR-2) — neutralized `validate_entry(&entry)` gate in `detect_replace_all_overcap_candidate` | rebuild + `test_MAJOR2_*` | **FAILED** — canonical sealed-and-truncated; the exact EC-022 destructive bypass the fix closes |
| **Mutation B** (MAJOR-1) — injected regressed re-check-before-staging ordering | rebuild + `test_MAJOR1_*` | **FAILED** with `SealedShardAlreadyExists` (not `SealWriteFailed`) — proves the ordering test pins stage→re-check, not just re-check existence |
| MAJOR-1 code trace | read `publish_sealed_shard` | order is `stage_temp_file` → `reclaim_identity_still_safe` (adjacent) → `remove_file` → `publish_staged_temp_file`; no durable write between re-check and unlink |
| MAJOR-1 leak check | staged-temp cleanup on both early returns + `publish_staged_temp_file` internal cleanup "regardless of outcome" | no leak on any path |
| Doc-site corrections | grep for adjacency claims | all four cycle-3 sites corrected (`shard_manager.rs` re-check comment, `reclaim_identity_still_safe` doc, Finding #8 test doc, README:46) |
| NIT-1 constant fact-check | `0x0100` semantics | comment now correct: `O_NOFOLLOW` on macOS/BSD, `O_NOCTTY` on Linux/Android (Linux `O_NOFOLLOW` = `0o400000`); empirically consistent with `test_N4` passing on darwin |
| Sibling-site sweep (TD-VSDD-060) | `execute_tiers` signature `+shard_gate_precheck_result` | all callers (7 test files + main) updated; compile + clippy + tests green confirm completeness |
| CI | `gh pr checks 824` | **not green** — build-dispatcher matrix (incl. windows-x64), cargo-host (both), bats-full-suite, bats-wave-handoff still pending; validate/policy-15/platforms-drift/deny-advisories/attestation-gate/SAST pass |

Working tree restored to pristine `27bbc6f8` after every mutation; `git status --porcelain` empty.

---

## Claimed-Fixed Items — Independent Confirmation

**MAJOR-1 (`e9941b4b`) — TOCTOU re-check adjacency restored — CONFIRMED FIXED, mutation-verified.**
Execution order in the 0-byte-reclaim branch is now `stage_temp_file` → `reclaim_identity_still_safe` → `remove_file(sealed_path)` → `publish_staged_temp_file`, with the re-check genuinely adjacent to the unlink and no durable write intervening (Finding #2's staging now precedes the re-check). (a) Adjacency confirmed. (b) All four stale doc sites corrected — the re-check comment, `reclaim_identity_still_safe`'s doc, the Finding #8 test doc, and README:46 now describe stage-first / re-check-immediately-before-unlink. (c) `test_MAJOR1_reclaim_identity_recheck_runs_immediately_before_unlink_not_before_staging` pins the ordering via a FIFO fixture + forced call-#1 staging failure; Mutation B proves it load-bearing (regressed ordering yields `SealedShardAlreadyExists`, tripping the assertion). (d) Both early-return paths clean up `staged_tmp_path`, and `publish_staged_temp_file` removes the temp regardless of outcome — no leak.

**MAJOR-2 (`0d91a4c3`) — EC-022 bypass on the catch-point-(i) leg closed — CONFIRMED FIXED, mutation-verified.**
`validate_entry(&entry)` is called in `detect_replace_all_overcap_candidate` immediately after `find_matching_entry` resolves a match, gating the destructive `reconcile_post_write_replace_all_overcap`→`execute_roll` path (returns `None` before it). On `Err` it emits a `tracing::warn!` (fail-open-but-never-silent, consistent with the F-C2-P1-005 arm above) and returns `None`. `validate_entry` genuinely rejects `artifact_path = "."`/`"./"`/empty via `EmptyArtifactPath` (EC-022). `test_MAJOR2_*` drives the real `reconcile_replace_all_overcap_if_qualifying`, establishes both preconditions (matcher matches the EC-022 entry, validator refuses it), and asserts zero destructive side effects (canonical untouched, no sealed shard, no index). Mutation A proves it load-bearing — removing the gate truncates the canonical to 0 bytes, exactly as the finding described.

**MAJOR-3 (`6d3d96e7`) — PreToolUse byte-cap gate hoisted before the empty-tiers early return — CONFIRMED FIXED.**
(a) `shard_cap_precheck(&payload, &project_cwd)` is called in `main::run` before the early-return guard, which is widened to `shard_gate_precheck_result.is_none() && sync_tiers.is_empty() && partition.async_group.is_empty()` — a fired verdict (`Some(_)`) no longer early-returns. The result is threaded into `execute_tiers` as a parameter and **consumed once** (no recompute — `shard_cap_precheck` is no longer called inside `execute_tiers`), which matters because the fired branch runs the destructive `execute_roll`. (b) Both stale comment sites corrected: `main.rs`'s catch-point-(i) rationale (MINOR-3 scope fix + the MAJOR-3 note now truthful post-hoist), the `shard_cap_precheck` doc in `executor.rs`, and the twin doc in `invoke.rs` all honestly describe the now-closed asymmetry — no comment asserts a guarantee that does not hold. (c) `test_MAJOR3_shard_cap_gate_fires_via_real_binary_when_no_plugin_matched` spawns the compiled binary with a zero-`[[hooks]]` registry and an EC-009-malformed `[[shard]]` config, asserting exit 2 — genuinely reaching the gate through `main::run` in the empty-plugin-set case. Verified passing.

**Cycle-3 MINOR/NIT sweep:**
- **MINOR-1 / MINOR-2 / NIT-4 (`878e6ab1`)** — `DuplicateArtifactStem` and missing/non-string `file_path` now emit `tracing::warn!`; `file_path` extraction reordered before the TOML parse. OK
- **MINOR-3 / MINOR-4 / MINOR-5 (`c0a57d75`)** — placement doc scoped honestly; coverage claim corrected to cite the real-binary test; MINOR-5 ordering trade-off documented + risk-assessed as accepted (no repo PostToolUse plugin re-reads its own artifact). OK
- **MINOR-5 test (`a4afa013`)** — `test_MINOR5_*` genuinely exercises a WASM validator observing the 0-byte canonical (not a no-op). OK
- **MINOR-4 test (`603eface`)** — real compiled-binary PostToolUse dispatch asserting sealed shard + 0-byte canonical. OK
- **NIT-1 (`361fa507`)** — `O_NOFOLLOW`/`O_NOCTTY` comment now factually correct. OK
- **NIT-2 (`02dfd02f`)** — symlink-rejection doc corrected (any symlink rejected on Unix by `O_NOFOLLOW`; target-size caveat scoped to non-unix fallback). OK
- **NIT-3 (`9d749546`)** — ancestor-probe doc states inconclusive `metadata` treated as "absent". OK
- **MINOR-6 / NIT-5 (`27bbc6f8`)** — Cargo.toml comment corrected (step (b) uses `write_exclusive`/`stage_temp_file`, not `write_atomic`; dep justified by (c)/(d)); NIT-5 accepted-with-revisit-trigger. OK
- **MINOR-7 (PR body staleness)** — **NOT addressed** (see MINOR-N2).

---

## Findings

### MINOR-N1 (NEW) — MAJOR-3's hoist places the destructive `execute_roll` before a fallible, now-unnecessary `build_engine()`, so a fired shard-gate Block can be silently downgraded to exit 0

`crates/factory-dispatcher/src/main.rs` (`run`)

The hoist moved `shard_cap_precheck` — whose fired branch runs `execute_roll` (seal + truncate-to-0, a completed filesystem mutation) — to before the widened early-return. When the gate fires (`Some(Block/Error)`) but both tier groups are empty, control now passes the early-return and reaches `build_engine()`. On `build_engine()` failure the function `return Ok(0)` — **dropping the already-fired Block verdict** (exit 0 instead of 2), even though the roll has already mutated the canonical. Claude Code would then apply the original over-cap write against the freshly-truncated canonical rather than being blocked-and-retried.

New interaction: pre-MAJOR-3 the roll lived inside `execute_tiers` (after `build_engine`), so a `build_engine` failure meant the roll never ran — consistent. Post-hoist the roll runs first, so roll-happened / verdict-dropped can diverge.

Why MINOR, not MAJOR: `build_engine()` (wasmtime `Engine::new` over a static `Config`) effectively never fails outside OOM, is not attacker-influenced, and is a pre-existing global fail-open that skips all hooks. The seal completes before `build_engine`, so no data is lost — a Write self-heals next dispatch; an Edit surfaces a not-found at CC level; `emit_dispatcher_error` leaves telemetry. Not a merge-blocker, but a real robustness/clarity defect the placement rationale does not mention.

**Suggested fix:** when both tier groups are empty, translate the `shard_gate_precheck_result` verdict to the exit code directly (or gate/move `build_engine()`) so a fired native verdict is never gated behind an unrelated fallible engine build — building a WASM engine to run zero plugins is also pure waste on that path.

### MINOR-N2 (= cycle-3 MINOR-7, STILL OPEN) — PR description is stale against the reviewed head

The PR body still cites tip `68796531` for its "workspace-green" / "454/454" evidence and the unchecked CI checkbox, still cites `current head 27d2f5ec` in the finding-disposition section (head is now `27bbc6f8`, 12 commits later), and still hedges the review pointer as "…or a sibling `pr-review-cycle2.md`". None of the eight cycle-3 remediation commits touch the PR body (code/test only), so this was never addressed. Refresh SHAs, test evidence, finding table, and CI checkbox before merge.

**Disposition:** resolved by the pr-manager's cycle-4 PR-body edit (dispatched alongside this review) — current-head references refreshed to `27bbc6f8`, Test Evidence refreshed with cycle-4's verified counts, cycle-2 pointer resolved unambiguously, and a Cycle 3 + Cycle 4 findings-disposition section added.

---

## Gate note — CI is not green at `27bbc6f8`

Pass: `validate`, `policy-15-attestation-location`, `platforms-drift`, `deny-advisories`, `attestation-gate-non-vacuity-controls`, `SAST (Semgrep)`. **Pending:** `build-dispatcher` (all five platforms), `cargo-host` (ubuntu + macos), `bats-full-suite (linux)`, `bats-wave-handoff (macos)`. `build-dispatcher (windows-x64)` is the merge-critical long pole — the only job executing the `#[cfg(windows)]` `is_genuinely_missing` tests (N-2 fix). Must be green before merge. (The `precompact-routing.bats` TC-AC004/TC-EC001 pre-existing flake was not observed and is not treated as a finding.)

---

## Checklist Summary

| # | Item | Result |
|---|---|---|
| 1 | Diff coherence | PASS — all changes scoped to `factory-dispatcher` + demo evidence |
| 2 | Description accuracy | MINOR-N2 (stale PR body — cycle-3 MINOR-7 still open; resolved by pr-manager's cycle-4 body edit) |
| 3 | Test coverage | Strong; MAJOR-1/2 mutation-verified; MAJOR-3 + catch-point-(i) covered by real-binary tests; MINOR-N1 build_engine-failure sub-path untested (accepted, rare) |
| 4 | MAJOR-1 re-verify | PASS — adjacency restored, 4 doc sites fixed, ordering test load-bearing, no temp leak |
| 5 | MAJOR-2 re-verify | PASS — `validate_entry` gates the destructive path, warn on Err, regression test load-bearing |
| 6 | MAJOR-3 re-verify | PASS — gate hoisted before early-return, consumed once, comments honest, empty-plugin-set real-binary test green |
| 7 | New sweep findings | MINOR-N1 (fired verdict droppable on `build_engine` failure); no MAJOR |
| 8 | fmt / clippy / commit hygiene | PASS — clean; no `Co-Authored-By`; `Claude-Session:` trailer per convention |
| 9 | CI | NOT green — build-dispatcher matrix (incl. windows-x64), cargo-host, bats still pending (merge gate) |

**Bottom line:** APPROVE on the code — the three MAJOR remediations are real and mutation-verified, and no MAJOR/blocking issue surfaced in the fresh sweep. Before merge: land CI green (windows-x64 especially). MINOR-N1 (don't gate a completed destructive roll's verdict behind `build_engine`) is non-blocking but should be routed to implementer for a follow-up fix — surfaced to the orchestrator rather than fixed by this review or by the pr-manager dispatch that requested it (out of scope for this dispatch, which was explicitly PR-body-edit + verdict only, no code changes, no merge).
