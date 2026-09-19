---
document_type: adr
adr_id: ADR-053
status: proposed
date: 2026-09-19
subsystems_affected: [SS-01, SS-04, SS-07, SS-10]
supersedes: null
superseded_by: null
---

# ADR-053: STORY-INDEX.md Fuel-Bounded Sharding — Extending the Layer-2 Shard-Cap Mechanism to the Story Catalog

> Design-only ADR. Pipeline is PAUSED per POLICY 22. No code, hook, or `.factory/` content
> change follows from this dispatch. This ADR requires human review before any
> implementation story is dispatched, and its own activation (§Decision 6) is explicitly
> sequenced BEHIND the cluster-5 / BC-1.18.010–011 POLICY 22 ratification already in flight —
> it does not request priority over that ratification.

## Context

`.factory/stories/STORY-INDEX.md` at HEAD (`git -C .factory show HEAD:stories/STORY-INDEX.md`)
is 548,497 bytes / 912 lines, covering ~220 story rows across 25 `## Epic E-NN` blockquote
sections. Every `Edit`/`Write` to it fuel-exhausts the dispatcher's 20,000,000 WASM fuel budget
(`factory_dispatcher::invoke::DEFAULT_FUEL_CAP`) on three `on_error = "block"` PostToolUse
validators — `validate-input-hash`, `validate-factory-path-root`, `validate-template-compliance`
— captured this session as `plugins_run=34 block_intent=true FUEL_EXHAUSTED fuel_cap=20000000`.
24 of the 34 registered PostToolUse plugins hit fuel timeout on this file; only these three are
configured `block` rather than `continue`, so they are the ones that stop the agent.

**Root cause, verified this session, not hypothesized.** All three blocking validators already
contain explicit early-exit routing that SKIPS `STORY-INDEX.md` by design:
`plugins/vsdd-factory/hooks/validate-input-hash.sh` exits 0 on any `*INDEX.md` path before doing
any hashing work; `validate-template-compliance.sh` and (the non-blocking)
`validate-story-bc-sync.sh` both exit 0 on any `*STORY-INDEX*` path. Despite this, all three
still exhaust fuel on this file. That is only possible if the fuel cost is paid BEFORE the
script's own case-pattern routing runs — i.e., during `INPUT=$(cat)` + `jq -r
'.tool_input.file_path'` marshaling of the PostToolUse JSON payload inside the WASM sandbox
(`hook-plugins/legacy-bash-adapter.wasm`, `crates/hook-plugins/legacy-bash-adapter/`, running
under `wasmtime` with fuel consumption enabled via `config.consume_fuel(true)` in
`factory_dispatcher::engine::build_engine`). For a `Write`, that payload embeds the full new file
content; for an `Edit`, it embeds `old_string`/`new_string`. Fuel is burned in proportion to
bytes the bash+jq interpreter processes, not to which branch of script logic eventually
executes. This means NO hook-script edit can fix this — three of the blocking hooks already do
the "right" thing internally and still fail. Only reducing the bytes marshaled per edit (i.e.,
sharding the artifact) addresses the actual mechanism. It also means simply raising the fuel cap
further (the ADR-042 precedent) is not durable here: this session's own evidence is that a 529 KB
file still exhausted a 20,000,000 cap, and STORY-INDEX.md's frontmatter `changelog:` array has
been growing by several dense entries per fix-burst for months — any fixed cap is a race against
unbounded growth that the growth eventually wins.

**Direct, already-merged precedent exists for exactly this problem class**, for a sibling
artifact with the same shape of failure — `BC-INDEX.md` (664,715 bytes at HEAD). CAP-043
("Artifact Sharding Layer 2: Size-Triggered Shard Rotation for Cycle Append-Logs and BC-INDEX
Structured-Catalog Sharding") was delivered as S-25.02 clusters 1–4, all merged on `develop`:

- **BC-1.18.005** (cluster 1, `fff5e4cc`): a NATIVE (non-WASM) `shard_cap_precheck` /
  `shard_cap_gate_check` in `factory_dispatcher::executor` + `factory_dispatcher::shard_manager`,
  driven by a `[[shard]]` array read from `.factory/shard-config.toml`. It performs a
  `stat()`-only byte-size check on every `Edit`/`Write`/`MultiEdit` against a registered
  artifact's `shard_cap_bytes` ceiling. Because this is native dispatcher-process code with no
  WASM sandbox, it "cannot exhaust a fuel budget because it has none" (BC-1.18.005 Postcondition
  2, verbatim). `shard_cap_bytes` is derived from four calibrated constants —
  `PRACTICAL_FUEL_CEILING`, `WORST_CASE_FUEL_PER_BYTE`, `MAX_SINGLE_RECORD_BYTES`,
  `SAFETY_MARGIN` — via
  `shard_cap_bytes <= (PRACTICAL_FUEL_CEILING / WORST_CASE_FUEL_PER_BYTE) - MAX_SINGLE_RECORD_BYTES - SAFETY_MARGIN`
  (BC-1.18.005 §Postconditions), explicitly marked "provisional... pending F4 harness
  calibration" in the BC's own title.
- **BC-1.18.006** (cluster 2, `0959e34b`): the roll-BEFORE-write mechanism — when
  `projected_size > shard_cap_bytes`, the check triggers a roll sequence before the write is
  allowed to land, so the artifact never actually crosses its cap on disk.
- **BC-1.18.007 / BC-1.18.008** (cluster 3, `08ad44b5`): `ShardShape::FrontmatterChangelogArray`
  — mechanism A, a governed ONE-TIME migration that moves a frontmatter `changelog:` YAML
  array's historical entries into a `<stem>-amendment-history.md` sidecar, with byte-for-byte
  content-preservation, an independent census, and crash-atomic staging + verify + atomic-replace.
  This shape is generic: it is keyed only on the presence of a `changelog:` array in frontmatter,
  not on any BC-INDEX-specific structure.
- **BC-1.18.009** (cluster 4, `ebd16f79`): `ShardShape::Flat` — mechanism B1, size-triggered
  rotation into sequential numbered shard files (today's intended consumer is cycle append-logs
  such as `burst-log.md`).
- **BC-1.18.010 / BC-1.18.011** (cluster 5, IN FLIGHT, explicitly OUT OF SCOPE for this ADR per
  instruction): mechanism B2, a `BC-INDEX`-SPECIFIC per-subsystem body-table split
  (`shards/BC-INDEX-SS-NN.md`, keyed on the already-authoritative BC-S-prefix→SS-NN mapping)
  with a top-level shard-manifest for zero-lookup first-level addressing, plus a second-level
  manifest for the two subsystems (SS-05, SS-06) that are themselves oversized. This is gated
  behind POLICY 22 human ratification (ADR-052 v1.13 lists 5 open sign-off items) and is not
  touched by this ADR.

This entire apparatus is built and unit-tested
(`crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs`,
`bc_1_18_006_roll_test.rs`, and siblings) but **DORMANT in this repository today**:
`.factory/shard-config.toml` does not exist in the committed tree (confirmed by search), so no
artifact — including `BC-INDEX.md` itself — is actually shard-cap-gated in production yet.
STORY-INDEX.md already HAS an `-amendment-history.md` sidecar
(`.factory/stories/STORY-INDEX-amendment-history.md`, 328,336 bytes, already registered in
`artifact-path-registry.yaml`), but it was populated by a ONE-TIME MANUAL extraction under an
explicit POL-3 exception (D-1149), not by the native mechanism. Its own header states verbatim:
*"Growth policy: per-cycle rotation (durable fix tracked in the follow-up write-path story). git
log -p is the authoritative archive."* STORY-INDEX.md is exactly the artifact that follow-up
story was meant to durably fix; S-25.02 was authored to be that follow-up (see
`.factory/stories/S-25.02-artifact-sharding-layer2.md`), and it built the generic machinery
first (clusters 1–4) before specializing to BC-INDEX (cluster 5) — STORY-INDEX was never itself
onboarded as a `[[shard]]` consumer.

**STORY-INDEX.md's mass has two independent drivers**, mirroring why BC-INDEX.md needed BOTH a
changelog mechanism (A) and a body mechanism (B1/B2) rather than one:

1. **Frontmatter `changelog:` array growth** — unbounded, dense, per-burst narrative entries.
   This is exactly what `ShardShape::FrontmatterChangelogArray` already handles generically; no
   new shape is needed for this driver.
2. **Body-section growth** — unlike BC-INDEX's homogeneous per-BC-row table, STORY-INDEX's body
   is 25 `## Epic E-NN — ...` blockquote sections, each holding dense inline per-story delivery
   narrative (this is where the reported single-row S-25.02 mass — 64.7 KB — actually lives: not
   in one table cell, but scattered across the `## Epic E-25` blockquote's inline per-story
   prose plus the many changelog entries that reference `S-25.02`). No existing `ShardShape`
   fits this: `Flat` assumes homogeneous append-only records; `FrontmatterChangelogArray`
   assumes a YAML array; `B2` is hard-coded to BC-INDEX's BC-S-prefix→SS-NN addressing and is
   out of scope. STORY-INDEX needs a third, generic shape keyed on its own natural partition —
   Epic ID (`E-NN`).

## Decision

1. **Reuse, don't reinvent, the trigger/roll gate.** Once `.factory/shard-config.toml` exists,
   register `.factory/stories/STORY-INDEX.md` as a `[[shard]]` entry, reusing the EXISTING
   native `shard_cap_precheck` / `shard_cap_gate_check` (BC-1.18.005) and roll-before-write
   mechanism (BC-1.18.006) verbatim. No new trigger/roll control-flow code is written — that
   machinery is artifact-agnostic already.

2. **Reuse the existing changelog shape.** Apply the EXISTING `ShardShape::FrontmatterChangelogArray`
   (mechanism A) to STORY-INDEX.md's frontmatter `changelog:` array, targeting its
   already-registered sidecar `.factory/stories/STORY-INDEX-amendment-history.md`. This converts
   the one-time D-1149 manual extraction into the durable, repeatable, native mechanism its own
   header called for — zero new path-registry work for this piece.

3. **Add ONE new, generic `ShardShape` variant for the body: `PerSectionBodySplit`.** Given a
   config-declared section-heading match pattern (`^## Epic (E-[0-9]+)`) and a target directory
   (`.factory/stories/index/`), this shape splits each matched section into its own shard file
   `.factory/stories/index/E-{epic-id}.md`, leaving in the root file only: frontmatter,
   `## Status Summary`, and a thin per-epic manifest table (Epic ID, Name, `story_count`, shard
   file path, byte size). The name is deliberately generic (not `B2` or `BC-INDEX`-flavored):
   it is parameterized purely by a heading regex and a capture group, so any future
   `## <Label> <ID>`-sectioned artifact can reuse it without a new shape.

4. **Root-file size target: ≤150 KB, hard ceiling 200 KB.** Each per-epic shard file is capped
   independently at the SAME `shard_cap_bytes` ceiling as the root. An epic whose own section
   exceeds the cap gets second-level sub-sharding by epic+story-range, mirroring B2's SS-05/SS-06
   second-level-manifest precedent (documented extension point; not needed at today's sizes —
   the largest existing epic section, E-25, is 35 KB).

5. **One-time migration via the ADR-052-sanctioned path, not Edit/Write.** Bringing the CURRENT
   548 KB file under cap requires a governed one-time migration, structured exactly like
   BC-1.18.008/011: a new native entry point (`run_mechanism_c_story_index_section_split`,
   `factory_dispatcher::shard_manager`, sibling to `run_mechanism_a_backfill_split`) that
   performs content-preservation verification, an independent census (every epic section and
   every story row lands in exactly one output file, zero duplicated/dropped stories), staging +
   verify + atomic-replace, and fail-loud rollback on any discrepancy — invoked through the
   `Bash` tool with the ADR-052 narrowly-scoped allowlist entry, never through `Edit`/`Write`, so
   the migration itself never touches the WASM fuel-bound hook chain.

6. **Activation is sequenced behind, not ahead of, cluster-5's POLICY 22 ratification.**
   `.factory/shard-config.toml` creation and this migration's execution both wait for the same
   human-ratification event already gating BC-1.18.010/011. This ADR extends that queue by one
   item; it does not request priority over cluster-5 and no code or `.factory/` content changes
   as a result of this ADR alone.

## Rationale

**Why reuse the native gate rather than build a parallel one.** BC-1.18.005/006's `stat()`-only
precheck is fuel-free by construction — it is not a WASM plugin, so it structurally cannot
exhaust a fuel budget regardless of file size. A STORY-INDEX-specific parallel gate would
duplicate exactly this property for zero benefit and would create two independent code paths for
the identical hazard class — the kind of duplication TD-VSDD-060 (sibling-site sweep discipline)
exists to prevent.

**Why a new shape rather than force-fitting `B2`.** BC-1.18.010 is titled and scoped explicitly
as "BC-INDEX Per-Subsystem Body-Table Sharding," keyed on the BC-S-prefix→SS-NN mapping that
only BC-INDEX has. STORY-INDEX has no such mapping; its natural partition is Epic ID. Reusing
`B2`'s name or its BC-INDEX-specific invariants (e.g., zero-lookup BC-S-prefix addressing) for a
structurally different key would be a naming and invariant mismatch, and would wire this ADR's
STORY-INDEX design to `B2`'s implementation, which is itself still mid-adversarial-review
(cluster-5, POLICY 22 OPEN) — an unstable foundation this ADR should not depend on. A new,
narrowly-generic `PerSectionBodySplit` shape is both correctly scoped for this artifact and
reusable beyond it.

**Why ≤150 KB, not just "smaller than today."** This session's own evidence: a 529 KB file
still exhausted a 20,000,000 fuel cap. Because fuel cost is driven by bytes the bash/jq
interpreter marshals from the PostToolUse payload (see §Context), not by script complexity, and
because `WORST_CASE_FUEL_PER_BYTE` is itself only a provisional, not-yet-calibrated constant
(BC-1.18.005's own title says so), the safe posture is a large margin below the last observed
failure point, not an incremental trim. 150 KB is >3.5x smaller than the still-failing 529 KB
case — margin for (a) BC-1.18.005's calibration uncertainty, (b) other plugins consuming fuel in
the same 34-plugin PostToolUse batch, and (c) a single `Edit`'s `old_string`+`new_string` payload
riding on top of the file's own size. This number is a design-time placeholder consistent with
BC-1.18.005 Postcondition 4's formula shape; it must be superseded by that BC's own calibrated
`shard_cap_bytes` once its F4 harness produces real constants — it is not a competing authority.

**Why migration-first, not hook-logic-first.** The root-cause finding in §Context (fuel
exhaustion happens during payload marshaling, before any script's own routing logic runs) means
editing hook scripts cannot fix this: three of the blocking hooks already skip STORY-INDEX.md
internally and still fail. Only reducing the bytes that must be marshaled per edit — i.e.,
sharding — addresses the actual mechanism.

## Consequences

### Positive

- Zero new fuel-budgeted code: the trigger/roll/gate machinery is 100% reused from BC-1.18.005/006,
  already merged and unit-tested.
- STORY-INDEX.md and BC-INDEX.md converge on one operational model (one `shard-config.toml`, one
  native gate, one migration-CLI pattern) instead of two independent large-artifact-resilience
  schemes.
- Bounds future growth structurally: once shard-capped, the native precheck rolls BEFORE a write
  that would exceed cap, so this failure class cannot silently reoccur the way it did between
  D-1149's one-time slim and today.
- `STORY-INDEX-amendment-history.md` and its `artifact-path-registry.yaml` entry are already
  correctly shaped as the mechanism-A target — no churn needed for that piece.

### Negative / Trade-offs

- Introduces a third `ShardShape` variant (`PerSectionBodySplit`) — genuinely new Rust code in
  `factory_dispatcher::shard_manager`, not pure reuse, requiring its own unit tests and
  BC-5.39.001 3-CLEAN adversarial review, comparable in scope to BC-1.18.009 (B1).
- Every hook/tool that assumes "STORY-INDEX.md is one file" must become shard-aware (7 identified
  in §Enumerated Hook and Tooling Impact) — a real propagation surface, not a config-only change.
- Adds a second governed one-time migration (alongside BC-1.18.008/011) to the human-ratification
  queue already gating cluster-5 under POLICY 22 — this ADR lengthens that queue by one item and
  explicitly does not claim priority over cluster-5.
- Root-level readers/writers (story-writer, state-manager dispatch prompts) must learn to consult
  the per-epic manifest before editing an epic's content — a workflow change, tracked as a
  propagation item, not a blocker to this ADR.

### Status as of 2026-09-19

Proposed. No code, config, or `.factory/` content has been changed by this ADR.
`.factory/shard-config.toml` still does not exist; `STORY-INDEX.md` remains unsharded and
remains fuel-blocked on direct `Edit`/`Write` today. Per §Decision 6, this ADR's own activation
is explicitly sequenced behind cluster-5/POLICY 22 ratification.

## Enumerated Hook and Tooling Impact

Every hook that reads or validates `STORY-INDEX.md`, and what each needs once sharding lands.
All seven are `legacy-bash-adapter.wasm`-hosted bash scripts (`plugins/vsdd-factory/hooks/*.sh`)
except `validate-artifact-path`, which is a compiled native Rust WASM plugin
(`hook-plugins/validate-artifact-path.wasm`). Per POLICY 21 ("no new shell scripts going
forward"), any NEW validation logic this migration requires must be added as native Rust
(a `shard_manager`-owned census/parity check, or a new dedicated WASM plugin crate under
`crates/hook-plugins/`), not as a new `.sh` file.

| Hook | Type | Current behavior on STORY-INDEX.md | Change needed |
|------|------|-------------------------------------|----------------|
| `validate-input-hash` | legacy-bash-adapter (bash) | Skips (`*INDEX.md` case-exit) before any hashing. | **None.** Root file stays `*INDEX.md`-suffixed, stays skipped. New shard files (`stories/index/E-NN.md`) do NOT match `*INDEX.md`; if they carry `inputs:` frontmatter they WILL be hash-checked — desired (drift detection is valuable on generated content), and safe (each shard is well under cap). Design note: shard files should NOT declare `inputs:` unless they are genuinely independently-authored; the migration tool should omit it or mark `input-hash: [live-state]`. |
| `validate-factory-path-root` | legacy-bash-adapter (bash) | Generic `.worktrees/` path check, artifact-agnostic. | **None.** Applies unchanged to any new `.factory/stories/index/E-NN.md` path. |
| `validate-template-compliance` | legacy-bash-adapter (bash) | Skips (`*STORY-INDEX*` case-exit). | **1-line hook-script change**: extend the skip-case list to also match `*stories/index/*.md` (or `E-[0-9]*.md` under that directory), treating epic shard files as index-class generated content — consistent with how `STORY-INDEX-amendment-history.md` is already exempted via the same case arm. Without this, every future epic-shard edit needs its own template, which is unnecessary overhead for generated/derived content. |
| `validate-story-bc-sync` | legacy-bash-adapter (bash) | Skips (`*STORY-INDEX*` case-exit); otherwise matches `*STORY-*.md`. | **None.** New shard filenames (`E-26.md`) do not match the `STORY-*` trigger pattern at all — the hook never fires on them regardless of the skip-case. |
| `validate-index-self-reference` | legacy-bash-adapter (bash) | Only triggers on `.factory/cycles/*/INDEX.md` or `.factory/cycles/*/burst-log.md`. | **None — confirmed non-applicable.** `STORY-INDEX.md` lives at `.factory/stories/`, not `.factory/cycles/*/`; this hook has never fired on it and does not need updating for sharding. |
| `validate-count-propagation` | legacy-bash-adapter (bash) | Triggers directly on `STORY-INDEX.md` (BASENAME match); also reads `STORY-INDEX.md` as a fixed sibling file whenever `ARCH-INDEX.md`/`BC-INDEX.md`/`VP-INDEX.md`/`STATE.md`/`PRD.md` is edited. | **None required IF the per-epic `story_count` stays authoritative in the root manifest** (§Decision 3/4 design already does this). The hook keeps comparing the root file's counts against siblings exactly as today; it never needs to open the per-epic shard files. If a future change moves count authority into the shards, this hook would need a companion Rust/native parity check (POLICY 21 forbids a new `.sh`) — flagged as a design constraint, not exercised by this ADR's chosen design. |
| `validate-state-pin-freshness` | legacy-bash-adapter (bash) | Triggers on `STATE.md` edits; reads STORY-INDEX.md's frontmatter `version:` field only, via an `awk` pass that `exit`s on first match. | **None.** This reads a handful of frontmatter lines regardless of body size; the `story_index_version` pin continues to track the ROOT file's own `version:` field, which is unaffected by where the body content lives. |
| `validate-artifact-path` (PreToolUse gate for all `.factory/` writes) | Native Rust WASM (`hook-plugins/validate-artifact-path.wasm`) | Reads `plugins/vsdd-factory/config/artifact-path-registry.yaml`; blocks writes to any `.factory/` path matching no registered pattern. | **Config-only** (no code change — the plugin is registry-driven): add new `artifact_type` entries for `.factory/stories/index/E-{epic-id}.md` and `.factory/shard-config.toml` (see §Artifact Path Registry Additions). Until these are registered, this is the PreToolUse gate that would reject the migration's own output writes — registry update must land in the SAME burst as (or before) the migration is run. |

**New governance surface required (not an existing-hook change):** a shard-manifest ↔
shard-file parity check — the root manifest's per-epic `story_count`/pointer must match the
actual content of `.factory/stories/index/E-NN.md`. Consistent with POLICY 21, this MUST be
implemented as native Rust (either as part of `factory_dispatcher::shard_manager`'s own
read-time census reuse, mirroring B2's manifest-consistency checks, or as a new
`crates/hook-plugins/` WASM plugin), not as a new bash script.

## Artifact Path Registry Additions

Two new entries in `plugins/vsdd-factory/config/artifact-path-registry.yaml`, both
`enforcement_level: block` (canonical location) consistent with every existing entry in that
file:

```yaml
  # ── Story Index Sharding (ADR-053) ───────────────────────────────────────
  - artifact_type: story-index-epic-shard
    canonical_path_pattern: ".factory/stories/index/E-{epic-id}.md"
    description: Per-epic body shard for stories/STORY-INDEX.md (PerSectionBodySplit shape, ADR-053 §Decision 3) — carries the epic's delivery narrative split out of the root catalog
    enforcement_level: block

  - artifact_type: shard-config
    canonical_path_pattern: ".factory/shard-config.toml"
    description: "[[shard]] registry consumed by shard_cap_precheck/shard_cap_gate_check (BC-1.18.005/006) — declares every shard-cap-gated artifact, its shape, and its shard_cap_bytes ceiling"
    enforcement_level: block
```

`story-index-amendment-history` (mechanism A's target) is already registered
(`plugins/vsdd-factory/config/artifact-path-registry.yaml`, "Story Index Sharding" §Behavioral
Contracts section preceding the Epics section) and needs no change.

## Fuel-Safe Migration Mechanism

The one-time migration MUST NOT go through `Edit`/`Write` on the 548 KB source file — that is
the exact operation being fixed. The mechanism, mirroring BC-1.18.008/011 exactly:

1. **Pre-migration:** the `artifact-path-registry.yaml` additions above land first (a small,
   ordinary Edit — the registry file itself is tiny and unaffected by this problem).
2. **Migration binary invocation:** `cargo run -p factory-dispatcher -- migrate story-index
   --section-split` (or the equivalent subcommand form established by ADR-052 §Decision 2),
   invoked via the `Bash` tool under the ADR-052 narrowly-scoped `.claude/settings.json`
   allowlist entry — never `Edit`/`Write`, so no PostToolUse hook (fuel-bound or not) ever
   receives the 548 KB payload as a tool-call argument.
3. **Content-preservation + census, native (no WASM, no fuel):** the binary (a) parses the
   source file's 25 `## Epic E-NN` sections and frontmatter `changelog:` array using ordinary
   Rust file I/O (not a WASM-sandboxed interpreter), (b) writes each epic's content to its own
   `.factory/stories/index/E-NN.md` staging file, (c) writes the trimmed root (frontmatter +
   `## Status Summary` + per-epic manifest) to a staging path, (d) runs an independent census:
   every story ID that appeared in the source appears in EXACTLY ONE output file, zero dropped,
   zero duplicated — mirroring BC-1.18.008/011's census design.
4. **Atomic publish:** staging files are `rename(2)`d into place only after the census passes;
   on any census failure the migration aborts loud, leaves the source file untouched, and deletes
   its own staging output (fail-loud rollback, no partial state).
5. **Post-migration:** the root `STORY-INDEX.md` is now ≤150 KB. Ordinary `Edit`/`Write` on it
   (and on each small `E-NN.md` shard) proceeds through the normal hook chain without fuel
   exhaustion, because the payload each hook now marshals is bounded by `shard_cap_bytes`, not
   by the corpus's total historical narrative mass.
6. **`.factory/shard-config.toml` registration:** the `[[shard]]` entry for STORY-INDEX (and,
   separately, BC-INDEX once cluster-5 lands) is added, activating BC-1.18.005/006's
   roll-before-write precheck so this class of regrowth-past-cap cannot recur silently.

## Bootstrap Sequencing (E-26 / STORY-INDEX Sharding Prerequisite)

Epic `E-26` ("Post-rc.25 Hook Hardening," 5 stories `S-26.01`–`S-26.05`, target `rc.26`) and its
story files already exist on disk as untracked drafts but are NOT YET registered in
`STORY-INDEX.md` — and registering them (an epic-table entry plus five story rows) is itself an
ordinary `Edit`/`Write` against the SAME 548 KB fuel-blocked file this ADR fixes. This is a real
bootstrapping order dependency, and it must be resolved explicitly rather than left as an
implicit assumption:

- **STORY-INDEX sharding is registered as `S-25.07`, not `S-26.06`.** It belongs to the existing
  epic `E-25` ("Validation Integrity and Large-Artifact Resilience"), not `E-26`. It shares
  capability `CAP-043`, subsystem `SS-01`, and the `BC-1.18.NNN` BC family with S-25.02's
  clusters 1–4, and its `PerSectionBodySplit` shape is a direct extension of that same body of
  work — not a hook-hardening defect fix, which is `E-26`'s distinct, orthogonal scope (four
  specific GitHub issues: PostToolUse mutation races, `validate-factory-path-staging` false
  positives, `pr-manager` dispatch-mode awareness, and a crate removal). Filing it under `E-26`
  would dilute that epic's `rc.26`-release-vehicle cohesion and misattribute the work's actual
  lineage.
- **Sequencing recommendation: `S-25.07` is dispatched and merged BEFORE `E-26`/`S-26.01`–`05`
  are registered in `STORY-INDEX.md`.** This is the conservative, durable choice: once `S-25.07`
  lands, registering `E-26` and its five stories becomes an ordinary small `Edit` against an
  already-lean root file — no special-casing needed, and no second one-time POL-3-style
  exception is required. The cost is that `E-26`/rc.26 registration (not implementation — the
  story specs already exist and can be authored/reviewed in parallel) waits on `S-25.07`'s own
  migration landing.
- **Explicitly rejected fallback:** registering `E-26`/`S-26.01`–`05` FIRST via a second one-time
  POL-3 human-approved exception (mirroring D-1149), ahead of `S-25.07`. This was considered and
  rejected as the DEFAULT path because it repeats exactly the non-durable pattern §Alternatives
  Considered rejects below — it would grow the fuel-blocked file further (adding 5 new story
  rows + 1 epic section) before the durable fix lands, working directly against `S-25.07`'s own
  goal. It remains available as a human-directed exception ONLY if `rc.26`'s timeline cannot
  absorb waiting for `S-25.07`, and only with the same explicit logging discipline D-1149 used.
- Both `S-25.07`'s registration into `STORY-INDEX.md` and — once it lands — `E-26`'s
  registration are themselves subject to the SAME fuel-exhaustion problem in the interim: the
  human-ratification gate in §Decision 6 means `S-25.07` cannot be registered via ordinary
  `Edit`/`Write` either until POLICY 22 clears for this class of change. This is an acknowledged,
  unavoidable chicken-and-egg property of fixing a fuel-blocked file using the fuel-blocked
  file's own change-control process, and is the reason §Fuel-Safe Migration Mechanism step 2
  routes through the ADR-052 `Bash`-tool allowlist rather than `Edit`/`Write` even for
  `S-25.07`'s OWN registration row.

## Alternatives Considered

- **Option: Raise the fuel cap further (ADR-042 precedent).** Rejected as insufficient alone:
  ADR-042's raise already addresses `validate-cross-site-correspondence`'s specific hazard, and
  this session's evidence shows even 20,000,000 fails on a file well within STORY-INDEX's
  current growth trajectory. A fixed cap is a race against unbounded growth; sharding bounds the
  artifact instead of chasing it. This does not contradict ADR-042 — that raise remains correct
  for its own artifact; it is cited here only to explain why the same lever does not durably
  solve this artifact's problem, and `E-26`'s own "fuel-cap release carry (ADR-042)" item is a
  complementary, not competing, piece of work.
- **Option: Repeat the ad hoc manual slim (the "548→378 KB" attempt this task's brief
  references).** Rejected as non-durable: manual slims are not repeatable, are not gated by an
  automatic precheck (so the file can silently regrow past budget before anyone notices — which
  is exactly what happened after D-1149's one-time extraction), and — per this task's own
  verification — were not confirmed to actually clear the fuel threshold, since they targeted
  changelog/history mass while leaving the larger per-epic body narrative untouched.
- **Option: Force-fit STORY-INDEX onto the in-flight `B2` mechanism as a second consumer.**
  Rejected: `B2`'s BC-S-prefix→SS-NN addressing has no analogue in STORY-INDEX's data model, and
  coupling this ADR's design to `B2`'s still-unratified, still-mid-adversarial-review
  implementation (cluster-5, POLICY 22 OPEN) would make STORY-INDEX sharding dependent on a
  moving target explicitly out of scope for this dispatch.
- **Option: Route ALL epic-body narrative into individual story files, eliminating epic-level
  sections from STORY-INDEX entirely (no `E-NN.md` shards at all).** Rejected as the sole
  mechanism: STORY-INDEX's epic sections answer "what wave/dependency order is this epic in, and
  what happened across its stories" — a cross-story view no single story file can hold.
  Eliminating the view loses information the corpus needs, not just bytes. The correct fix
  bounds the view's SIZE via sharding, not its existence — consistent with per-story delivery
  narrative that is genuinely story-specific still being pushed down into the individual
  `S-NNN.md` file rather than duplicated in the epic shard, which the migration design (§Fuel-Safe
  Migration Mechanism step 3) already does by construction (the epic shard receives the epic's
  own text; it does not invent new per-story detail that belongs in the story file).

## Source / Origin

- **This session's direct verification (2026-09-19):**
  `git -C .factory show HEAD:stories/STORY-INDEX.md | wc -c` → `548497`;
  `git -C .factory show HEAD:stories/STORY-INDEX.md | wc -l` → `912`; dispatcher trace evidence
  `plugins_run=34 block_intent=true FUEL_EXHAUSTED fuel_cap=20000000` on `validate-input-hash`,
  `validate-factory-path-root`, `validate-template-compliance`.
- **Root-cause evidence (this session):** `plugins/vsdd-factory/hooks/validate-input-hash.sh`
  (`*INDEX.md` skip case); `plugins/vsdd-factory/hooks/validate-template-compliance.sh` +
  `validate-story-bc-sync.sh` (`*STORY-INDEX*` skip case) — all three still exhaust fuel despite
  these early exits.
- **Fuel/engine mechanics:** `factory_dispatcher::invoke::DEFAULT_FUEL_CAP` (`= 20_000_000`);
  `factory_dispatcher::engine::build_engine`'s `config.consume_fuel(true)` call.
- **Precedent mechanism (BC-INDEX.md, same problem class):**
  `factory_dispatcher::shard_manager` (`ShardEntry`, `ShardShape::Flat`,
  `ShardShape::FrontmatterChangelogArray`, `shard_cap_precheck`);
  `.factory/specs/behavioral-contracts/ss-01/BC-1.18.005.md` through `BC-1.18.011.md`;
  `.factory/specs/architecture/decisions/ADR-051-layer-2-two-mechanism-size-triggered-shard-rotation-append-logs-and-bc-index-sharding.md`;
  `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md`;
  `.factory/stories/S-25.02-artifact-sharding-layer2.md`.
- **STORY-INDEX's own acknowledged deferred-fix marker:**
  `.factory/stories/STORY-INDEX-amendment-history.md` header — "Extracted 2026-09-02 under
  one-time POL-3 exception (D-1149)... Growth policy: per-cycle rotation (durable fix tracked in
  the follow-up write-path story)."
- **Artifact path registry (current state):**
  `plugins/vsdd-factory/config/artifact-path-registry.yaml` (story-index,
  story-index-amendment-history, story-spec, epic entries — all pre-existing and unaffected by
  this ADR except for the two additions in §Artifact Path Registry Additions).
- **Governance gates:** `.factory/policies.yaml` POLICY 22 (human-ratification channel
  extension, D-970 Codification 2) and POLICY 21 (`no_new_shell_scripts`) — both cited in
  §Decision 6 and §Enumerated Hook and Tooling Impact respectively; `.factory/STATE.md` current
  cycle marker (pipeline PAUSED, read but not modified by this dispatch).
- **Bootstrap context:** `.factory/stories/epics/E-26-post-rc25-hook-hardening.md` and
  `.factory/stories/S-26.01`–`S-26.05-*.md` (untracked drafts, not yet registered in
  `STORY-INDEX.md` as of this ADR).
