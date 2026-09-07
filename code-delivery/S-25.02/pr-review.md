# PR #818 — Fresh-Eyes Review (cycle 4, closing convergence check)

**PR:** #818 — `feat(S-25.02): cluster 1 — cap formula + native shard-cap trigger (BC-1.18.005)`
**Branch:** `feature/S-25.02-cap-trigger` → `develop`
**Head reviewed:** `f870af0b099efdaa4bf6f9612c6cb91d90602236`
**Verdict:** **APPROVE** — 0 BLOCKING, 1 SUGGESTION, 3 NIT

> Prior cycles (superseded; full text in this file's git history on `factory-artifacts`):
> cycle 1 — REQUEST_CHANGES (4 blocking); cycle 2 — REQUEST_CHANGES (2 blocking B-1/B-2,
> 2 major M-1/M-2); cycle 3 — APPROVE (0 blocking, 2 suggestion, 3 NIT). Cycle 3's S-2
> (`./`-prefix path-normalization) and S-3 (stale M-2 test comment) are verified closed below.

Reviewed against the actual current diff at the head SHA (`gh pr diff 818`, `git show <sha>:<path>`),
not against the fix-burst narrative or any prior review's claims. Finding severity has decayed
monotonically across four passes (4 blocking → 2 blocking + 2 major → 0 blocking + 2 suggestion →
0 blocking + 1 suggestion), consistent with a genuinely converging review.

**Scope reviewed:** 22 files, +5,949/−1. Source: `Cargo.toml` / `Cargo.lock`, `executor.rs`,
`lib.rs`, new `crates/factory-dispatcher/src/shard_manager.rs` (4,135 lines incl. tests), new
`crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs` (1,131 lines). Remainder is
demo evidence under `docs/demo-evidence/S-25.02/cluster-1-cap-trigger/`.

**CI:** all 17 checks pass (cargo-host ×2, bats-full-suite, bats-wave-handoff, bats-darwin-leg,
build-dispatcher ×5, SAST/Semgrep, deny-advisories, attestation-gate-non-vacuity-controls,
platforms-drift, policy-15-attestation-location, validate). `mergeStateStatus: CLEAN`.

---

## 1. Cycle-3 findings — both CONFIRMED CLOSED

### S-2 (`./`-prefix path normalization) — CLOSED, verified mechanically

`shard_manager.rs:782` `path_falls_under_or_equals` now filters `Component::CurDir` out of **both**
operand component vectors before the suffix/prefix comparison. Verified against the head commit's
own diff (`git show f870af0b`): the production change is exactly the two
`.filter(|c| *c != std::path::Component::CurDir)` insertions and nothing else.

The fix introduces no new defect in its own mechanism:

- Filtering **both** sides means normalization works whether the `./` appears on the registered
  path or on the target path — not just the operator-facing case the commit message describes.
- The `registered.len() <= target.len()` (suffix) and `registered.len() < target.len()` (prefix)
  guards still short-circuit ahead of the slice indexing, so the shorter post-filter vectors
  cannot produce an out-of-range panic.
- It does not regress the cycle-2 B-1 stem-vs-path tests: filtering is a no-op on any path with no
  `CurDir` component, which is every fixture in the B-1 suite
  (`/registered/dir/decision-log.md`, `/registered/dir`).

**The regression test is load-bearing, not vacuous.** `shard_manager.rs:1981`
`test_BC_1_18_005_B1_path_falls_under_or_equals_curdir_prefix_matches_same_as_without` asserts both
the equality-with-baseline **and** a standalone
`assert!(path_falls_under_or_equals(target, with_curdir_prefix))`. A mutually-broken baseline
(both spellings matching nothing) cannot make it pass.

### S-3 (stale M-2 test comment) — CLOSED

The comment on `test_BC_1_18_005_B2_read_changelog_item_count_oversized_file_fails_loud` no longer
describes the removed `stat()`-only rejection path; it now correctly names the M-2 bounded
`Read::take(MAX_CHANGELOG_TARGET_READ_BYTES + 1)` mechanism on an already-open handle.

---

## 2. New findings

### F-1 — [SUGGESTION] `artifact_path` values normalizing to an empty component list become stem-only wildcards; S-2 newly widened this set

| Field | Value |
|-------|-------|
| Severity | suggestion |
| Category | coherence |
| File | `crates/factory-dispatcher/src/shard_manager.rs:782` (`path_falls_under_or_equals`) |

When `registered` is empty after `CurDir` filtering, `suffix_match` evaluates
`target[target.len()..] == []` → **unconditionally true**, so the entry matches *every* file
sharing its `artifact_stem`.

That is precisely the stem-only blast radius the cycle-2 B-1 finding was raised to close. The
field's own doc at `shard_manager.rs:125-153` states the case explicitly: `artifact_path` is
required, never `#[serde(default)]`, because "this repository alone has 426 files sharing the
`STATE` stem, 99 sharing `lessons`, 98 sharing `burst-log`…" — a stem-only entry would route every
one of those through the matched entry's `validate_entry` / cap gate.

`artifact_path = ""` already hit this before S-2 (`Path::new("")` yields zero components), so the
underlying hole is **pre-existing**, not introduced here. What S-2 changes is that `"."` and `"./"`
now join it: pre-S-2 they yielded `[CurDir]`, which could never suffix-match a real target (so
those entries matched **nothing**); post-S-2 they normalize to empty and match **everything** with
the stem. The direction flipped from fail-closed to fail-open, which is the less-safe direction
under this module's own blast-radius doctrine.

`validate_entry` never inspects `artifact_path` at all, and could not help here regardless — under
the v1.12 MATCH-FIRST restructure it runs only *after* a match has already been resolved.

**Suggestion.** One-line close, either:

```rust
// in path_falls_under_or_equals, after filtering:
if registered.is_empty() {
    return false; // a degenerate artifact_path is never a wildcard match
}
```

or a structural non-empty `artifact_path` check surfaced as a new `ShardConfigError` variant at
match time, consistent with the module's fail-loud-on-config-defect posture everywhere else.

**Why not blocking.** No `[[shard]]` config file is committed anywhere in the repository, so
`shard_cap_precheck`'s `Path::exists()` probe short-circuits every dispatch and the gate is inert
in production today. Reaching this requires an operator to author a degenerate `artifact_path` in a
config that does not yet exist. Cleanly deferrable to the BC-1.18.006 cluster-2 PR, which touches
this exact function.

### F-2 — [NIT] Stale `ShardRegistry::load()`-time claims contradicted by the file's own section header

| Field | Value |
|-------|-------|
| Severity | nit |
| Category | description |
| File | `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs:365-366, 392-393, 409-410, 437-438` |

Four sites still assert that EC-009 / EC-011 fail loud "at `ShardRegistry::load()` time" / via "the
`HookResult::Error` `ShardRegistry::load` returns":

- L365-366 inline comment (`fail-loud MissingShape at ShardRegistry::load() time`)
- L392-393 assertion message
- L409-410 inline comment (`fail-loud InvalidLowWaterMark at ShardRegistry::load() time`)
- L437-438 assertion message

The section header 40 lines above (L313-323) already states the post-v1.12 truth correctly:
`ShardRegistry::load` is structural-TOML-parse-only, both fields are `Option`-typed and deserialize
fine, and fail-loud enforcement happens in `validate_entry` at entry-match time. The inline
comments contradict the header directly above them.

Same de-stale class as commits `df948674`, `93f93a81`, `b2b3a947` already on this branch.

### F-3 — [NIT] Two broken cross-references in the m5 test; stale guard count in the module header

| Field | Value |
|-------|-------|
| Severity | nit |
| Category | description |
| File | `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs:8-24, 714` |

- **L714:** "Same EC-013 malformed config as the AC-023 test above" — no test in this file is named
  AC-023 (`grep 'AC-023'` returns this comment and nothing else). The intended referent is
  `test_BC_1_18_005_P2001_cap_exceeds_ceiling_block_reason_names_artifact_stem_and_failure_kind`
  at L582. The config is also **not** the same: m5 uses stem/path `decision-log`, P2001 uses
  `over-cap-log`.
- **L8-24 module header:** "It applies two cheap, real guards" and "The two negative-control
  tests" — there are **three** guards (`event_name`/`EventType` classification at
  `executor.rs`, then tool-name, then config-presence) and **three** negative controls; the
  PostToolUse control at L820 landed after the header was written.

### F-4 — [NIT] EC-019 "missing required non-Option field" fixture does not isolate the field it names

| Field | Value |
|-------|-------|
| Severity | nit |
| Category | coverage |
| File | `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs:1087-1093, 1094-1102, 1121` |

The fixture omits **`artifact_path` as well as `shard_cap_bytes`**. Both are non-`Option` with no
`#[serde(default)]` (`shard_manager.rs:153` and `:170`), and `artifact_path` is declared first in
the struct, so the `toml::de::Error` surfaced is almost certainly about `artifact_path` — while
the comment (L1087-1093) and the assertion message (L1121) both name `shard_cap_bytes` as the
field under test.

The assertion remains load-bearing for EC-019's actual contract (a structural parse failure blocks
regardless of match); only the field attribution is imprecise. Dropping the `shard_cap_bytes` line
alone, keeping `artifact_path`, would make the fixture match its own description.

### F-5 — [NIT] Three `reaches_native_gate` assertion messages state arithmetic no assertion can verify

| Field | Value |
|-------|-------|
| Severity | nit |
| Category | coverage |
| File | `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs:121-226` |

The messages cite specific cap comparisons — "5,000 <= 49,152" (L152), "45,004 <= 49,152"
(L181/185), "48,000 <= 49,152" (L219/223) — but `shard_cap_gate_check`'s Flat arm emits
`tracing::warn!` and falls through to `HookResult::Continue` on a fired trigger
(`shard_manager.rs:1512-1532`); it never constructs `Block` in this cluster. So `exit_code == 0`
holds identically for an over-cap payload, and no assertion in these three tests can fail on the
arithmetic its message describes. What they genuinely pin is "the dispatch reaches the gate and
the matched entry passes `validate_entry`" — which is exactly what their *names* say.

The scope boundary itself is legitimate and openly disclosed: `shard_manager.rs:1613-1621` states
plainly that no test asserts an outcome for the trigger-FIRES branch at gate level, because
BC-1.18.006 / BC-1.18.009 own that observable outcome. The trigger *decision* is genuinely covered
at the `size_trigger_fires` / `item_count_trigger_fires` unit level. Only the message text
overclaims — the same narrative-attestation-vs-mechanical-evidence smell the project polices under
D-449(a) / META-LEVEL-24. Trimming the arithmetic from the three messages closes it.

---

## 3. Already-closed per prior adjudication — deliberately NOT re-flagged

| Item | Disposition |
|------|-------------|
| **B3** — `replace_all` multiplicity in the projected-size formula | Deferred to BC-1.18.006 per product-owner spec ruling |
| **m1** — `effective_shard_cap_bytes` has no live caller in `shard_cap_gate_check` | Intentional config-time / F4-harness helper per BC-1.18.005 v1.10 adjudication (finding F-C1-P4-003, "Reading (A) CONFIG-TIME/HARNESS-HELPER adjudicated over Reading (B) RUNTIME"); rationale recorded inline at `shard_manager.rs:857-871` |
| **m5** — `plugins_run` counts the synthesized shard-gate outcome | Consistent with the three sibling native/sentinel outcome sites (`spawn_blocking` join-error, `payload serialize`, `plugin load failed`); rationale recorded in `shard_gate_block_outcome`'s doc comment |

---

## 4. Verified clean — what this pass actually checked

Recorded explicitly per the no-rubber-stamping requirement.

**Diff coherence (checklist 1).** Every changed file traces to BC-1.18.005. The two new
dependencies are both justified inline in `Cargo.toml` (`serde_norway` for the frontmatter
`changelog:` count with a precedent cite to `last-amended-migrate/src/yaml_guard.rs`;
`vsdd-hook-sdk` to reuse the canonical three-variant `HookResult` rather than invent a parallel
result type) and are reflected in `Cargo.lock` with no other lock churn. `vsdd-hook-sdk` as a
path-only dep with no `version` key differs from the sibling entries in the same file, but
`factory-dispatcher` is `publish = false`, and path-only is the dominant convention for this
crate elsewhere in the workspace — **not a finding**.

**Description accuracy (checklist 2).** The PR body's corrected latency accounting (finding B4)
matches the module doc at `shard_manager.rs:22-45` and the actual code: `Path::exists()` probe is
O(1), but a committed config means one bounded whole-file TOML parse per Edit/Write/MultiEdit
dispatch even for non-matching targets. The body does not overclaim zero-cost.

**Executor wiring.** The gate runs before the registry-driven tier loop (Invariant 1). Both
`HookResult::Error` and `HookResult::Block` flip `block_intent` **and** push a synthesized
`PluginOutcome` into `all_outcomes`, so `main.rs::extract_block_info`'s scan over
`per_plugin_results` has something to find (the F-C1-P2-001 fix). `Continue` is a genuine no-op.

**Error handling.** No `unwrap()` / `expect()` on any production path. Named `thiserror` variants
throughout; every fail-loud message carries `artifact_stem` plus an EC marker. Both `Option`
destructures that follow `validate_entry` (`entry.shape` at L1368, `entry.n` at L1557) use
fail-loud `let`-`else` rather than assuming the just-validated invariant — correct defensive
posture for a future refactor.

**Arithmetic safety.** `compute_shard_cap_bytes` saturates on both subtractions;
`projected_size_edit` branches on delta sign to avoid unsigned underflow;
`validate_low_water_mark` widens to `i128` at the `>= N` boundary; `item_count_trigger_fires` uses
`saturating_add`. The EC-015 (non-finite / non-positive divisor) and EC-017 (tiny-positive divisor
saturating the raw pre-cast division) guards both run *before* the cap-vs-formula comparison, so
neither divisor-door can defeat it.

**Bounded read.** `read_changelog_item_count` uses a single `take(MAX + 1)` on an already-open
handle, closing both the TOCTOU window and the non-regular-file (FIFO reporting `len() == 0`) gap
that a `stat()`-then-`read_to_string` shape would leave. The closing-fence scan is line-anchored
and correctly rejects `----`, `---foo`, and in-block-scalar lines beginning `---`.

**Test coverage (checklist 3).** 17 integration tests drive the real `execute_tiers` →
`shard_cap_precheck` → `ShardRegistry::load` → `shard_cap_gate_check` stack, including
block-reason surfacing (P2001 ×3), EC-018 sibling-scoping ×2, EC-019 ×2, the B-2 missing-
`file_path` fail-loud, the m5 `plugin_version` propagation, and a PostToolUse negative control.
Independently verified as **non-vacuous** (contrary to one delegated reading): the
`PC1_non_mutating_tool_name_bypasses_native_gate` control at L268 passes `tool_input: {}`, so
removing the tool-name guard would reach the B-2 missing-`file_path` fail-loud and yield
`exit_code == 2` — the assertion genuinely constrains the guard it names. Cap arithmetic in the
fixtures independently recomputed: `8_000_000 / 106.36 → 75_216`, `− 16_384 − 8_192 = 50_640`, so
`shard_cap_bytes = 49_152` correctly passes and `100_000` correctly trips EC-013.

**Demo evidence (checklist 4).** `docs/demo-evidence/S-25.02/cluster-1-cap-trigger/` contains 5
`.gif` + 5 `.webm` + 5 `.tape` sources + `README.md` with a full AC/EC → clip mapping. Recordings
drive real `cargo test` runs against unmodified source (no hand-typed output). The README is
candid about the one thing it cannot show — the `tracing::warn!` advisory — because no
`tracing_subscriber` is wired anywhere in the workspace; it substitutes the real call site plus the
passing boolean-decision test. Honest evidence, not a `.txt` placeholder.

**Commit quality (checklist 5).** Conventional Commits format with story ID throughout
(`fix(S-25.02):`, `test(shard-manager):`, `docs(demo):`). No AI attribution in any commit message.

**Diff size (checklist 6).** Exceeds the 500-line flag threshold, but the excess is one new module
plus its test suite for a single BC cluster whose boundary is spec-recorded (BC-1.18.005 only;
BC-1.18.006 / BC-1.18.009 / BC-1.18.012 explicitly out of scope). Not a finding.

**Missing changes (checklist 7).** All six ACs claimed in the PR body (AC-001..AC-005, AC-023) have
corresponding implementation and tests. The interim `warn!` + `Continue` hand-off for a fired
trigger is disclosed in the PR body, the module header, and both trigger branches — an honest
cluster boundary, not silent swallowing.

**Dependency status (checklist 8).** ADR-051 merged; BC-1.18.004 active. No unmerged upstream PR
gates this one.

---

## 5. Recommendation

**Merge.** The suggestion and three NITs are all cleanly deferrable to the BC-1.18.006 cluster-2 PR,
which touches this exact module. None affects runtime behavior while no `[[shard]]` config is
committed. If the team prefers them closed here, F-1 is a one-line guard plus one test, and F-2
through F-5 are comment-text edits.
