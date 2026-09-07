# [S-25.02] Artifact Sharding Layer 2 — Cluster 1: Cap Formula + Native Size/Item-Count Trigger

**Epic:** S-25.02 — Artifact Sharding Layer 2 (BC-1.18.001..012)
**Mode:** feature (Feature Mode F4, cycle `v1.0-brownfield-backfill`)
**Convergence:** CONVERGED at BC-5.39.001 3-CLEAN (LOCAL adversary passes 10/11/12, D-1172) after 12 cluster-1 LOCAL adversarial passes total. PR-LEVEL review converged after 5 fresh-context review cycles (see Adversarial Review below).

![Tests](https://img.shields.io/badge/tests-3006%2F3006-brightgreen)
![Bats](https://img.shields.io/badge/bats-2234%2F2234-brightgreen)
![fmt](https://img.shields.io/badge/fmt-clean-brightgreen)
![clippy](https://img.shields.io/badge/clippy--D%20warnings-clean-brightgreen)

This PR delivers cluster 1 of the incremental-by-BC-cluster delivery of S-25.02 (per D-1170): the
native (non-WASM) dispatcher PreToolUse **shard-cap GATE + TRIGGER** for **BC-1.18.005**. It adds
`crates/factory-dispatcher/src/shard_manager.rs`, computing a `shard_cap_bytes` ceiling from four
calibrated config inputs, dispatching on artifact `shape` to either a `stat()`-only flat byte-size
trigger or an item-count trigger for the `"frontmatter-changelog-array"` shape, and wiring the
match-first `ShardRegistry::load()` (structural-parse-only) + pure `validate_entry` gate into
`executor.rs`'s PreToolUse path with fail-loud block-reason surfacing for load-time config defects
and graceful missing-file degradation.

**Cluster boundary (explicit, spec-recorded — not a gap):** this cluster owns the cap-formula and
the trigger-BOUNDARY decision only. The *observable* roll-before-write + `HookResult::Block` action
is owned by **BC-1.18.006** (cluster 2, not yet in scope). Today, a fired byte-size trigger on the
`"flat"` shape surfaces as `tracing::warn!` + `Continue` — an honest interim, not silent
swallowing (BC-1.18.005 Postcondition 3, "Ownership" text).

---

## Architecture Changes

```mermaid
graph TD
    PreToolUse["executor.rs::PreToolUse dispatch"] -->|Edit/Write/MultiEdit| ShardGate["shard_cap_precheck()"]
    ShardGate -->|config exists?| ConfigProbe["Path::exists() shard-config.toml (O(1))"]
    ConfigProbe -->|no| Continue1["Continue"]
    ConfigProbe -->|yes| Registry["ShardRegistry::load() — structural TOML parse"]
    Registry -->|match stem + path containment| Match["find_matching_entry(target_path)"]
    Match -->|None| Continue2["Continue — no validate_entry call"]
    Match -->|Some(entry)| Validate["validate_entry(entry) — fail-loud config checks"]
    Validate -->|invalid| BlockErr["HookResult::Error — fail-loud block_reason"]
    Validate -->|valid| GateCheck["shard_cap_gate_check() — shape dispatch"]
    GateCheck -->|shape=flat| FlatTrigger["bounded projected_size formula"]
    GateCheck -->|shape=frontmatter-changelog-array| ItemTrigger["item-count trigger, bounded read"]
    FlatTrigger -->|over cap| Warn["tracing::warn! + Continue (roll deferred to BC-1.18.006)"]
    FlatTrigger -->|under cap| Continue3["Continue"]
    ItemTrigger -->|over N| Warn
    ItemTrigger -->|under N| Continue3
    style ShardGate fill:#90EE90
    style Registry fill:#90EE90
    style Validate fill:#90EE90
    style GateCheck fill:#90EE90
```

<details>
<summary><strong>Architecture Decision Record — match-first entry validation (BC-1.18.005 v1.12) + path-containment hardening (v1.13)</strong></summary>

### ADR: Match-first (not eager-fail-fast) `[[shard]]` entry validation

**Context:** `ShardRegistry::load()` originally validated every `[[shard]]` config entry eagerly on
every dispatch whose config file existed — a single malformed sibling entry would block *every*
`Edit`/`Write`/`MultiEdit` in the repo, including edits to unrelated files.

**Decision:** `find_matching_entry` runs first (structural parse only, now also requiring path
containment via a required `artifact_path` field, not stem alone); `validate_entry` runs *only*
against the entry the current dispatch's target path actually matches.

**Rationale:** Blast-radius containment is this BC's own purpose. Path-only matching (added during
PR-level review convergence, findings B-1/S-2/F-1) closes the same class of blast-radius defect on
the directory axis that match-first closed on the sibling-entry axis: a stem-only match would let
an edit to any same-named file anywhere in the repo (measured: hundreds of collisions in this
repo's own test fixtures) trigger a registered artifact's gate.

**Consequences:**
- A single malformed sibling entry no longer blocks unrelated dispatches.
- A same-stem file in a different directory no longer matches.
- An `artifact_path` that normalizes to empty (`.`, `./`) is now rejected at config-validation
  time rather than vacuously matching every file sharing the stem.

</details>

---

## Story Dependencies

```mermaid
graph LR
    ADR051[ADR-051<br/>merged] --> S2502C1[S-25.02 cluster-1<br/>this PR]
    BC1_18_004[BC-1.18.004<br/>active] --> S2502C1
    S2502C1 --> BC1_18_006[BC-1.18.006 cluster-2<br/>owns roll/Block]
    S2502C1 --> BC1_18_009[BC-1.18.009<br/>item-count rotation contract]
    style S2502C1 fill:#FFD700
```

---

## Spec Traceability

```mermaid
flowchart LR
    BC[BC-1.18.005 v1.13<br/>Cap formula + native trigger] --> AC1[AC-001<br/>match-first + path-containment scoping]
    BC --> AC2[AC-002<br/>per-tool-semantics formula]
    BC --> AC3[AC-005<br/>item-count trigger + low_water_mark]
    BC --> AC4[AC-023<br/>load-time fail-loud cap-vs-formula]
    AC1 --> T1["match_first_* + path_containment_* tests"]
    AC2 --> T2["projected_size_* tests"]
    AC3 --> T3["item_count_trigger_* tests"]
    AC4 --> T4["load_time_* tests"]
    T1 --> S1["shard_manager.rs"]
    T2 --> S1
    T3 --> S1
    T4 --> S1
    S1 --> S2["executor.rs PreToolUse wiring"]
```

---

## Test Evidence

### Coverage Summary

| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
| Cargo workspace tests | 3006/3006 pass | 100% | PASS |
| Bats integration suite | 2234/2234 pass | 100% | PASS |
| `cargo fmt --check --all` | clean | clean | PASS |
| `cargo clippy --workspace --all-targets -- -D warnings` | clean | 0 warnings | PASS |

| Metric | Value |
|--------|-------|
| **New source module** | `crates/factory-dispatcher/src/shard_manager.rs` |
| **New integration test file** | `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs` |
| **Executor wiring diff** | `executor.rs` (PreToolUse-scoped only) |
| **Regressions** | 0 |
| **New regression tests added during PR-level review convergence** | 30+ across B-1/B-2/M-1/M-2/N-2/S-2/F-1 fixes, each with a load-bearing positive+negative control (TD-VSDD-059) |

---

## Holdout Evaluation

N/A — evaluated at wave gate (Feature Mode F4 does not run a per-cluster holdout pass; deferred to
the next wave-boundary gate per feature-mode-scoping-rules).

---

## Adversarial Review

| Pass | Scope | Findings | Status |
|------|-------|----------|--------|
| 1-9 | LOCAL adversary, cluster-1 cap-trigger diff | Multiple HIGH/MEDIUM/LOW | All fixed |
| 10-12 | LOCAL adversary | 0 findings | CLEAN 3/3 — **3-CLEAN CONVERGED (D-1172)** |
| PR cycle 1 | Fresh-eyes pr-reviewer | 4 BLOCKING, 1 MAJOR, 6 MINOR, 4 NIT | All BLOCKING/MAJOR fixed |
| PR cycle 1 | Cognitive-diversity code-reviewer | 1 MAJOR, 3 MINOR, 2 NIT | MAJOR fixed |
| PR cycle 2 | Fresh-eyes re-review | 2 BLOCKING (B-1 path containment incomplete, B-2 file_path silent default), 2 MAJOR | All fixed |
| PR cycle 2 | Code-reviewer re-verify | APPROVE, 1 new MINOR (test vacuity) | Fixed |
| PR cycle 3 | Fresh-eyes re-review | APPROVE, 1 suggestion (S-2 CurDir path-normalization gap) | Fixed |
| PR cycle 3 | Code-reviewer re-verify | APPROVE, 0 findings | — |
| PR cycle 4 | Fresh-eyes closure review | APPROVE, 1 suggestion (F-1 empty-path fail-open gap) | Fixed |
| PR cycle 4 | Code-reviewer closure review | APPROVE, 0 findings | — |
| PR cycle 5 | Final closure review (both reviewers) | APPROVE, 0 blocking, 1 non-blocking theoretical-completeness item deferred to cluster 2 | CONVERGED |

**Deliberately-not-fixed items, spec-adjudicated (not silent gaps):**
- **B3** (`replace_all` occurrence-multiplicity): explicitly deferred to BC-1.18.006 per product-owner ruling recorded in BC-1.18.005 v1.11 "Known formula gap."
- **m1** (`effective_shard_cap_bytes`): adjudicated as an intentional config-authoring-time helper, BC-1.18.005 v1.10 (F-C1-P4-003).
- **m5** (`plugins_run` counting): verified consistent with 3 sibling native/crash outcome sites.

Full convergence detail: `.factory/code-delivery/S-25.02/pr-review.md` (cycle-5 final content;
prior cycles' findings preserved in `factory-artifacts` git history).

---

## Security Review

See `.factory/code-delivery/S-25.02/security-review.md`. 4 findings (SEC-001..SEC-004), 0
CRITICAL/HIGH at any point. The two LOW findings (SEC-001 unbounded read, SEC-002 stem-only
matching) were fixed during PR-level review convergence and independently re-verified by multiple
fresh-context reviewers before merge.

---

## Risk Assessment & Deployment

### Blast Radius
- **Systems affected:** `factory-dispatcher` binary's PreToolUse hook path only (native code, no
  WASM plugin). No changes to any WASM hook plugin, no changes to `hooks-registry.toml`.
- **User impact if this trigger misfires:** worst case is a `tracing::warn!` log line (non-blocking)
  on the flat shape, or a fail-loud `HookResult::Error` block on a genuinely malformed `[[shard]]`
  config entry the current dispatch targets — scoped to that one artifact's edits only. No
  roll/rotation/write-blocking behavior exists yet in this cluster (owned by BC-1.18.006).
- **Data impact:** none — this cluster reads config + bounded file content; it performs no mutation
  of any tracked artifact.
- **Risk Level:** LOW — the gate is additive, PreToolUse-scoped, zero-cost-bypass for dispatches
  that don't match a `[[shard]]` entry, and has no live `[[shard]]` config committed yet.

### Performance Impact
| Metric | Before | After | Delta | Status |
|--------|--------|-------|-------|--------|
| Unmatched-path dispatch latency | baseline | baseline + O(1) `Path::exists()` probe | negligible | OK |
| Matched-path dispatch latency | n/a (new capability) | bounded read/arithmetic per dispatch when config exists | new, bounded (not zero-cost once a config exists — corrected from an earlier doc overclaim during review) | OK |

<details>
<summary><strong>Rollback Instructions</strong></summary>

**Immediate rollback:**
```bash
git revert <squash-merge-commit-sha>
git push origin develop
```

No feature flag exists for this gate — it is inert (zero-cost bypass) until a `[[shard]]` config
file is committed, which has not happened in this cluster. Rollback is a plain revert.

</details>

---

## Traceability

| Requirement | Story AC | Test | Status |
|-------------|---------|------|--------|
| BC-1.18.005 PC1 / Invariant 3 / EC-018 / EC-019 / EC-020 / EC-021 | AC-001 | match-first + path-containment Red Gate suite | PASS |
| BC-1.18.005 PC3 / EC-005 | AC-002 | per-tool-semantics projected-size tests | PASS |
| BC-1.18.005 PC5 | AC-003 | `effective_shard_cap_bytes` unit tests | PASS |
| BC-1.18.005 PC6 / PC7 | AC-004 | config-driven constants tests | PASS |
| BC-1.18.005 PC8 / EC-008/010/011/012/014/016 | AC-005 | item-count trigger + `low_water_mark` tests | PASS |
| BC-1.18.005 PC9 / EC-013/015/017 | AC-023 | load-time fail-loud cap-vs-formula tests | PASS |

Full spec: `.factory/specs/behavioral-contracts/ss-01/BC-1.18.005.md` (v1.13) ·
`.factory/stories/S-25.02-artifact-sharding-layer2.md`

Demo evidence: `docs/demo-evidence/S-25.02/cluster-1-cap-trigger/` (5 per-AC/EC VHS clips +
README, using anchor-based extraction per TD-VSDD-091).

---

## Demo Evidence

`docs/demo-evidence/S-25.02/cluster-1-cap-trigger/` contains 5 recordings (`.gif` + `.webm` +
`.tape` each), one per acceptance criterion / edge case, extracted via anchor-based (function-name
and unique-literal) `.tape` scripts rather than volatile line-number pins (TD-VSDD-091, corrected
during PR review from an earlier `sed -n 'NNN,NNNp'` line-pinned version):

| Clip | Coverage |
|------|----------|
| `AC-002a-under-cap-write-continue` | A `Write` under the configured cap proceeds with `Continue` — no false-positive block. |
| `AC-002b-over-cap-trigger-fires` | A `Write` over the configured cap fires the `tracing::warn!` trigger (roll/block deferred to BC-1.18.006), confirming the trigger boundary itself works. |
| `AC-023-malformed-config-fail-loud` | A malformed `[[shard]]` config entry produces a fail-loud `HookResult::Error` with a diagnostic `block_reason`, scoped to the entry it matches (match-first, not repo-wide). |
| `EC-014-missing-changelog-first-write` | A first-ever write to a not-yet-existing frontmatter-changelog-array artifact degrades gracefully to `Ok(0)`/`Continue` rather than blocking artifact creation. |
| `EC-018-match-first-blast-radius` | An edit to a file NOT matching any `[[shard]]` entry proceeds unaffected even when a sibling entry is malformed, confirming blast-radius containment. |

The README candidly discloses one evidence limitation rather than papering over it: the AC-002b
`tracing::warn!` clip cannot show the warning printed to the terminal because `tracing_subscriber`
is not wired into this workspace's binary output path — the trigger firing is instead confirmed via
the test suite's assertion on the `HookResult`/log-record content, not a terminal screenshot.

---

## AI Pipeline Metadata

<details>
<summary><strong>Pipeline Details</strong></summary>

```yaml
ai-generated: true
pipeline-mode: feature
factory-cycle: v1.0-brownfield-backfill
feature-phase: F4 (delta-implementation)
pipeline-stages:
  spec-evolution: completed
  incremental-stories: completed
  delta-implementation: completed
  local-adversarial-cascade: completed (BC-5.39.001 3-CLEAN, passes 10/11/12, D-1172)
  demo-evidence: completed
  pr-level-review: completed (5 cycles to convergence)
convergence-metrics:
  local-adversarial-passes: 12
  clean-streak: 3/3
  pr-review-cycles: 5
cluster-scope: "cluster 1 of 2 (BC-1.18.005 cap+trigger; BC-1.18.006 roll/Block deferred to cluster 2)"
generated-at: "2026-09-07"
```

</details>

---

## Pre-Merge Checklist

- [x] All CI status checks passing
- [x] No critical/high security findings unresolved
- [x] Rollback procedure is a plain `git revert` (no feature flag needed — gate is inert without a
      committed `[[shard]]` config)
- [x] Demo evidence recorded per-AC/EC (5 clips + README)
- [x] pr-reviewer + code-reviewer findings triaged to 0 blocking (5 review cycles to convergence)

---

https://claude.ai/code/session_01NFH8fjTgmsKWWcQbEYJ9xA
