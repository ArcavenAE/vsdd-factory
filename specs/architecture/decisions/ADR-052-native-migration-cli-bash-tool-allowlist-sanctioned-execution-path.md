---
document_type: architecture-decision-record
level: L3
version: "1.2"
status: proposed
producer: architect
timestamp: 2026-09-13T00:00:00Z
phase: F1
subsystems_affected:
  - SS-01
traces_to: .factory/specs/architecture/ARCH-INDEX.md
inputs:
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-dir-cluster5-F1-direction-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-adr052-cluster5-F1-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/research-adr-052-assumption-validation-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-adr052-v11-closure-2026-09-12.md
  - .factory/stories/S-25.06-append-log-backfill-split-executor.md
  - .factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md
  - .factory/cycles/v1.0-brownfield-backfill/s2502-cluster5-f1-delta-analysis.md
  - CLAUDE.md
  - .claude/settings.json
  - crates/factory-dispatcher/src/main.rs
  - plugins/vsdd-factory/hooks-registry.toml
input-hash: "593ef95"
# input-hash: run compute-input-hash --update at state-manager registration burst
---

# ADR-052: Native Migration CLI — Sanctioned Execution Path for Governed One-Time Shard Migrations (v1.2 Redesign)

## Status

PROPOSED — v1.1 NOT ratified per D-1216 (2nd Codex RATIFY-WITH-CHANGES verdict, 8 findings).
This v1.2 redesign addresses all 8 findings from the 2nd Codex closure review
(`adv-cv-adr052-v11-closure-2026-09-12.md`). Human POLICY 22 ratification required before this ADR
is treated as accepted. Cluster-5 TDD dispatch remains BLOCKED until ratification.

## Context

Two governed one-time shard migrations are specified in S-25.02:

1. **Mechanism-A append-log backfill-split** (BC-1.18.008) — wired and executed by S-25.06.
   `run_mechanism_a_backfill_split` exists in `shard_manager.rs` with zero production callers.
   S-25.06 adds the CLI entry point and executes the migration against the four live cycle
   append-log files.

2. **Mechanism-B2 BC-INDEX body-split migration** (BC-1.18.011) — cluster-5 of S-25.02.
   The `run_mechanism_b2_bc_index_split` function (to be authored in cluster-5 TDD) is the
   native migration entry point in `shard_manager.rs`.

Both migrations share a structural challenge: they perform filesystem writes against `.factory/`
paths but cannot satisfy two simultaneously imposed constraints as originally stated in S-25.06
Rule 7:

- **Constraint A:** The executor MUST be a native binary (not WASM; WASM fuel budgets are
  insufficient for multi-MB files per ADR-051 §Decision 1 / CLAUDE.md §WASM plugin fuel
  budgets).
- **Constraint B (original Rule 7 text):** The invocation must be "logged as an Edit/Write
  tool call" — implying it goes through the agent's Edit/Write surface that hooks validate.

These constraints are mutually exclusive: a native binary invoked from the shell writes to the
filesystem OUTSIDE the Edit/Write tool surface.

Three project-level governance constraints apply:

- **POL-3 / TD-FACTORY-HOOK-BYPASS-001 P0:** Agents must NOT use Python/sed/echo/shell to
  bypass the hook-validated `.factory/` write surface. This ADR declares a narrowly-authorized
  exception under POLICY 22 (§Decision 8); it does NOT claim the migration is "not a bypass."
- **D-449(a):** Mechanical gates in burst-log Dim-2 require literal shell execution with
  captured stdout, not pseudocode narrative.
- **F4 activation gate:** Both migrations are one-time operations that require explicit human
  authorization and must be executed at a known activation boundary — not triggered
  automatically.

**Correction from v1.0:** `.claude/settings.json` (project-level) currently contains ONLY
`enabledPlugins`; there are no existing Bash permission entries. The v1.0 claim about "existing
project Bash permissions" establishing precedent was false (confirmed by 1st Codex review and
research Q3).

**Dispatcher guard constraint:** `plugins/vsdd-factory/hooks-registry.toml`
registers 13 entries with `tool = "^Bash$"`, of which 9 are PreToolUse. A settings.json Bash
allowlist suppresses the Claude Code permission prompt but does NOT suppress any PreToolUse
hook — dispatcher hooks fire regardless of allow-rules. Any mechanism that invokes the migration
binary via the Bash tool must address the 9 PreToolUse `^Bash$` guards or the command will be
blocked before execution.

**Native admission gate constraint (v1.2 addition):** The dispatcher's native execution path
for Edit/Write/MultiEdit tools goes through `shard_cap_precheck` in `executor.rs` before any
registry plugin fires. A maintenance lock check placed only in a `^Bash$` registry guard
therefore does NOT cover native mutation paths (e.g., `execute_roll` via `shard_cap_gate_check`
at `shard_manager.rs`). A production-grade maintenance boundary requires a native check in the
dispatcher binary BEFORE `shard_cap_precheck`, covering all mutation tools.

**Option A viability:** Anthropic docs confirm hooks execute with full user permissions. The
dispatcher already executes native mutations as a PreToolUse side-effect via
`shard_cap_precheck` (in `executor.rs`) whose fired verdict routes to
`shard_manager::execute_roll` — a destructive seal-and-truncate operation — proving the
hook-internal native mutation pattern works end-to-end in this codebase. The v1.0 rejection of
hook-driven migration evaluated ONLY unconditional, detection-based ("automatically-triggered")
activation; explicitly-armed one-shot hook activation was not evaluated. This v1.1/v1.2 ADR
evaluates all three options honestly (§Rationale).

## Decision

### Decision 1 — Mechanism selection: one-time interactive Bash approval (Option B)

Governed one-time shard migrations are invoked via the **`Bash` tool with one-time interactive
human approval at F4 activation time — NO standing allowlist entry in settings.json.**

The agent invokes `{project-root}/target/release/factory-dispatcher migrate-bc-index` at the
F4 activation step. The Claude Code harness shows a permission prompt; the human approves once.
The migration runs, completes, and the permission expires — no entry is added to settings.json;
no standing permission persists. This is Option B from the three-way evaluation.

See §Rationale for the honest head-to-head evaluation of Options A (hook-driven), B
(one-time-interactive), and C (hardened standing allowlist). Option A is viable; its rejection
is on operational complexity grounds only. Option C fails F3 (standing permission ≠
activation authorization).

### Decision 2 — Binary placement for both migration entry points (unchanged from v1.0)

Both migration entry points live in the `factory-dispatcher` binary
(`crates/factory-dispatcher/`), co-located with `shard_manager.rs`. Rationale:

- `shard_manager.rs` already owns the migration algorithm for mechanism A
  (`run_mechanism_a_backfill_split`). B2's entry point (`run_mechanism_b2_bc_index_split`)
  follows the same co-location model.
- `last-amended-migrate` is BC-10.13.001-scoped; extending it for body-structure migrations
  requires a BC-10.13.001 amendment without commensurate benefit.
- `factory-dispatcher` already has a CLI dispatch layer; adding `migrate-bc-index` and
  `backfill-append-logs` subcommands is additive and consistent with the existing pattern.

### Decision 3 — Invocation: absolute-path-pinned, closed argument grammar, no standing allowlist

The agent MUST invoke the migration binary at its **absolute trusted path** derived from the
project root, not via shell PATH resolution:

```
{project-root}/target/release/factory-dispatcher migrate-bc-index
{project-root}/target/release/factory-dispatcher backfill-append-logs
{project-root}/target/release/factory-dispatcher migrate-bc-index --census
{project-root}/target/release/factory-dispatcher backfill-append-logs --census
```

The accepted argument grammar is CLOSED: exactly the above four forms, no additional flags,
no path arguments to migration targets (targets are config-driven), no shell metacharacters,
no compound commands. Any other argument form is REJECTED by the guard's pre-shell classifier
(§Decision 5) BEFORE the command reaches shell execution, and rejected again by the binary
with a non-zero exit if somehow bypassed.

No entry is added to `.claude/settings.json`. The one-time interactive approval at F4
activation IS the permission mechanism; it is temporally bound to the exact activation moment,
not pre-granted.

### Decision 4 — Activation authorization decoupled from permission grant (addresses 1st Codex F3 + 2nd Codex F4)

A standing permission grant MUST NOT serve as activation authorization. Authorization is a
separately-recorded approval bound to the specific migration, repository state, and readiness
checks. The manifest validation is split into two phases: a lightweight pre-lock check and a
full under-exclusion validation.

#### 4a — Armed-activation manifest structure

State-manager writes the manifest to `.factory/activation/migrate-bc-index-YYYY-MM-DD.json`
(or `backfill-append-logs-YYYY-MM-DD.json`) at the F4 human-directed activation step.

The manifest MUST contain:
- `activation_id`: unique UUID for this activation (used for crash-recovery lock correlation)
- `migration_id`: `"migrate-bc-index"` or `"backfill-append-logs"`
- `repo_root_sha`: the SHA of the factory-artifacts commit that was HEAD at approval time
- `approved_arch_index_sha`: the ARCH-INDEX committed SHA the config snapshot was generated
  from (§Decision 10; must equal live ARCH-INDEX SHA at approval time)
- `expected_total_bcs`: integer read from BC-INDEX frontmatter at approval time (B2 only;
  corroborates census oracle; validated under exclusion before any mutation)
- `approved_by`: `"human-F4-interactive"`
- `timestamp_utc`: ISO-8601 timestamp of manifest creation
- `expires_after_hours`: `24` (migration must complete within 24 hours of approval)

Writing the manifest creates the record of human intent. The manifest itself is a write to
`.factory/activation/` which is an allowed write target (§Decision 8).

#### 4b — Pre-lock manifest checks (lightweight, before lock acquisition)

Before acquiring the maintenance lock (§Decision 7), the binary performs only these checks:
1. Manifest file exists at the expected path — fails CLOSED if absent
2. `migration_id` field matches the running subcommand — fails CLOSED on mismatch
3. Manifest file is parseable (not corrupt) — fails CLOSED on parse failure

These lightweight checks prevent the binary from acquiring the lock on a clearly invalid
activation state, without requiring filesystem reads that could race with concurrent writes.

#### 4c — Under-exclusion validation (after lock acquisition)

After acquiring the maintenance lock (§Decision 7 — exclusive.lock held), the binary performs
the full validation before ANY filesystem mutation to the migration targets:
1. Re-verify manifest exists and `migration_id` still matches
2. Current UTC time is within `expires_after_hours` of `timestamp_utc` — fails CLOSED on
   expiry (prevents replay against a later repository state)
3. Current factory-artifacts HEAD SHA matches `repo_root_sha` — fails CLOSED on mismatch
4. `expected_total_bcs` matches current BC-INDEX frontmatter `total_bcs` — fails CLOSED on
   mismatch (detects intervening BC additions/removals since approval)
5. Three-way ARCH-INDEX parity check (§Decision 10): `config.arch_index_sha` ==
   `manifest.approved_arch_index_sha` == live ARCH-INDEX SHA — fails CLOSED if any pair
   diverges
6. All readiness preconditions per BC-1.18.011 §Preconditions pass

The lock acquisition is itself a filesystem write (creating `exclusive.lock`). This ordering
— lightweight pre-lock checks, then lock, then full validation under exclusion — is the
correct sequence for a cooperative maintenance boundary.

#### 4d — Pre-publication expiry recheck

Immediately before the PREPARED → COMMITTED transition (§Decision 7), the binary re-verifies
that the manifest has not expired. If more than `expires_after_hours` have elapsed since
`timestamp_utc`, the migration ABORTS: the PREPARED marker is deleted and the staged content
is discarded. This prevents a slow migration from publishing stale results.

#### 4e — Manifest consumption and recovery modes

After the migration reports success (COMMITTED phase marker written), the binary adds a
`consumed_at` timestamp to the manifest file (atomic overwrite of the manifest with the added
field). The manifest is then archived by state-manager to `.factory/migration-audit/` as part
of the durable audit record (§Decision 6). The binary never deletes the manifest directly.

Re-run behavior is governed by the migration state, NOT by manifest absence:
- **COMMITTED marker present**: migration already complete; exit 0 with `ALREADY_MIGRATED`
  status regardless of manifest state.
- **PREPARED marker present + manifest present + unexpired**: crash-recovery mode;
  `activation_id` in PREPARED must match manifest `activation_id` to proceed.
- **PREPARED marker present + manifest absent or expired**: the crash occurred AFTER the
  activation window. The binary ABORTS and requires a new activation manifest (new human
  approval). It does NOT proceed with potentially stale authorization.
- **No markers + manifest absent**: reject as unauthorized; exit non-zero.
- **`--census` flag**: read-only census of current BC-INDEX state; no lock required; no
  manifest required.

This eliminates the v1.1 contradiction where "absent manifests are rejected" conflicted with
"already-migrated reruns exit 0" — the COMMITTED marker governs idempotency, the manifest
governs authorization.

### Decision 5 — Dispatcher PreToolUse guard stack + native admission gate (v1.2 REDESIGNED)

#### 5a — Native maintenance admission gate (F1 — covers ALL mutation tools, pre-shard_cap_precheck)

The maintenance lock check MUST fire as the FIRST action in the dispatcher's execution path
for ALL mutation operations — before any registry plugin and before `shard_cap_precheck` in
`main.rs`. This is a native binary check, not a registry guard, because:

- `shard_cap_precheck` in `executor.rs` fires for Edit/Write/MultiEdit BEFORE the registry
  plugin chain. A `^Bash$` registry guard cannot protect against a native mutation that
  bypasses the registry entirely.
- The admission gate must therefore be implemented as a check at the top of
  `executor.rs`'s dispatch path, inspecting the mutation type and target path.

**Admission gate logic (native, pre-registry):**
1. If the operation is a mutation (Edit/Write/MultiEdit/Bash) AND the target path is under
   `.factory/specs/behavioral-contracts/` or `.factory/cycles/`:
2. Check for `exclusive.lock` at `.factory/migration-state/exclusive.lock`
3. If the lock exists: verify the PID in the lock file is still running (platform-specific
   process check). If the PID is alive, return E-MAINTENANCE immediately without executing
   the mutation or invoking any hook. If the PID is dead, the lock is stale — see §Decision 7
   for stale-lock recovery.
4. If no lock exists, proceed normally.

**Drain protocol for in-flight writers:**
The race window between "writer admitted before lock creation" and "migration starts" is
handled by the TOCTOU guard (§Decision 7 §7c): the migration re-verifies the source
fingerprint immediately before the first rename. Any write that completed between census and
TOCTOU check will cause the TOCTOU check to fail and abort the migration (not the writer).
This is the correct production-grade resolution: the migration aborts and requires re-activation,
rather than silently publishing potentially stale shard content.

**Downstream binary changes:** The `executor.rs` dispatch entry point must be amended to
invoke the admission gate check before the `shard_cap_precheck` call. This is an implementer
(cluster-5 TDD) deliverable.

#### 5b — Guard amendments for `^Bash$` PreToolUse guards

A settings.json allowlist suppresses the Claude Code permission prompt; it does NOT suppress
dispatcher PreToolUse hooks. The 9 `^Bash$` PreToolUse guards fire on every Bash tool call
regardless of settings.json allow-rules. Each must be addressed before the migration can
execute:

| Guard | Expected behavior on migration command | Required action |
|-------|----------------------------------------|-----------------|
| `block-ai-attribution` | Inspects Bash input for AI-attribution patterns; migration command contains none | No amendment needed |
| `check-factory-commit` | Detects multi-commit chain in `.factory/` git log; migration runs within a single burst | No amendment needed provided burst discipline is maintained |
| `destructive-command-guard` | Flags migration command as potentially destructive (writes to `.factory/`) | **MUST be amended** to recognize the absolute-path-pinned `factory-dispatcher migrate-bc-index` and `factory-dispatcher backfill-append-logs` commands after full-command pre-shell classification (§5c below) |
| `protect-secrets` | Scans Bash input for secret patterns; migration command contains no secrets | No amendment needed |
| `verify-git-push` | Only fires on `git push` patterns; irrelevant | No amendment needed |
| `validate-factory-path-staging` | Validates that Bash commands targeting `.factory/` paths do so through proper channels | **MUST be amended** with full-command pre-shell classifier (§5c) and lock-check; the guard MUST validate manifest content (not just presence) before passing the command |
| `verify-factory-lock-bash` | Checks factory lock state | Activation sequence MUST acquire factory lock before invoking migration; guard should then pass |
| `validate-heavy-op-delegation` | May flag migration as a heavy operation requiring orchestrator delegation confirmation | **MUST be amended** to allow the migration command when the armed-activation manifest is present and validated |
| `validate-unvalidated-mutation-marker-git` | Checks for unvalidated `.factory/` mutations staged via git | Devops-engineer factory-artifacts commit step handles staging via normal git; guard does not conflict with binary-level writes |

**Guard amendment deliverable:** devops-engineer amends `destructive-command-guard`,
`validate-factory-path-staging`, and `validate-heavy-op-delegation` in hooks-registry.toml
and the underlying hook scripts/plugins to recognize the governing absolute-path-pinned
migration command pattern, validated per §5c.

#### 5c — Full-command pre-shell classifier (F7 — guard-level, before shell execution)

**The migration binary CANNOT reject shell metacharacters or compound commands.** A binary
receives `argv` AFTER the shell has already parsed and executed the command string. A command
like `{project-root}/target/release/factory-dispatcher migrate-bc-index && rm -rf /` is split
by the shell into two separate commands before `argv` reaches the binary; the binary never
sees the `&& rm -rf /` part.

The pre-shell classifier MUST be implemented in the `validate-factory-path-staging` guard
(and mirrored in `destructive-command-guard`). This guard receives the full proposed Bash
command string from the tool input BEFORE shell execution. The classifier must:

1. **Exact match:** The entire command string must exactly match one of the four permitted
   forms from §Decision 3 (with `{project-root}` resolved to the actual absolute project
   root path, no trailing characters).
2. **Metacharacter rejection:** Reject if the command string contains: `;`, `&&`, `||`, `|`,
   `>`, `<`, `` ` ``, `$(`, `\n`, `\r`, or any whitespace beyond the single space separating
   the binary path, subcommand, and optional `--census` flag.
3. **Alternate path rejection:** The executable path must start with the canonical project
   root (resolved via `git rev-parse --show-toplevel` at guard load time) and end with
   `/target/release/factory-dispatcher`. Any deviation (e.g., `/tmp/factory-dispatcher`,
   a symlink outside the project tree, a relative path) must be rejected.
4. **Manifest content validation:** After command-string validation, the guard reads the
   armed-activation manifest and verifies: manifest parseable, `migration_id` matches the
   subcommand, manifest not expired, manifest `activation_id` is a valid UUID. Failure on
   any of these rejects the command with a descriptive error. Manifest PRESENCE alone is
   not sufficient.
5. **Executable digest verification:** The guard reads the SHA-256 of the binary at the
   resolved absolute path and compares it against the expected digest stored in
   `.factory/activation/factory-dispatcher.sha256` (written by devops-engineer at build
   time as part of the cluster-5 activation package). Mismatch rejects the command.

Negative tests demonstrating that unrelated destructive commands never inherit the exception
MUST be added (cluster-5 TDD scope):
- `{project-root}/target/release/factory-dispatcher migrate-bc-index && rm -rf /tmp/test` → REJECTED
- `/tmp/evil/factory-dispatcher migrate-bc-index` → REJECTED
- `{project-root}/target/release/factory-dispatcher migrate-bc-index --extra-flag` → REJECTED
- `{project-root}/target/release/factory-dispatcher` (no subcommand) → REJECTED (no match)

### Decision 6 — Audit trail: durable, tamper-evident record (NIST AU-9) (unchanged from v1.1)

Census-to-stdout is correctness evidence for D-449(a); it is NOT an adequate audit record.
NIST SP 800-53r5 AU-9 requires audit records to be tamper-evident and protected from
modification or deletion, independent of the current session. Two independent audit artifacts
are required:

1. **D-449(a) evidence (stdout capture):** The migration CLI MUST write a structured census
   report to stdout on completion (success or idempotent no-op). Minimum required fields:
   - Source file path(s) operated on
   - Pre-migration record count (from `total_bcs` oracle for B2; from known file count for A)
   - Independent-census record count (fresh enumeration of actual rows/entries)
   - Per-shard record counts and file paths (including sub-shards for SS-05/SS-06 in B2)
   - Content-preservation hash (SHA-256 of source body before migration)
   - Idempotency status (`FIRST_RUN` | `ALREADY_MIGRATED` | `RESUMED_FROM_CHECKPOINT`)
   - Migration binary version and build SHA
   - Exit code taxonomy (0 = success, non-zero with E-SHD-* error code on failure)

   State-manager captures the stdout verbatim as the D-449(a) burst-log Dim-2 attestation.

2. **Durable audit record (NIST AU-9):** Immediately after the migration completes (COMMITTED
   phase marker present), state-manager commits to factory-artifacts:
   - The captured census stdout as `migration-audit/migrate-bc-index-YYYY-MM-DD-census.txt`
     (or `backfill-append-logs-...`)
   - The armed-activation manifest (consumed copy from `.factory/activation/`)
   - The COMMITTED phase marker file
   - The migration binary's `--version` output

   The factory-artifacts commit hash is recorded in STATE.md's burst-log for the activation
   burst. This factory-artifacts commit is the tamper-evident record: it is immutable in git
   history and not subject to session compaction.

### Decision 7 — Crash-atomicity, lock ownership, and atomic publication (v1.2 REDESIGNED)

#### 7a — Lock ownership protocol (F3 — separate maintenance intent from OS advisory lock)

The maintenance lock is implemented as a **content-bearing lock file**, NOT a pure existence
lock:

- **Lock file path:** `.factory/migration-state/exclusive.lock`
- **Lock file contents:** JSON with fields `pid` (integer), `activation_id` (UUID matching
  the armed-activation manifest), `timestamp_utc` (ISO-8601 acquisition time)
- **Atomic acquisition:** Create the file with `O_CREAT | O_EXCL` (atomic on POSIX). On
  creation failure (file exists), proceed to recovery check below.
- **"Held" condition:** The lock is held if the file exists AND either:
  - The PID in the file is still running (checked via `/proc/<pid>/status` on Linux or
    equivalent on macOS), OR
  - The `activation_id` in the file matches the `activation_id` in a valid PREPARED marker
    (indicating a crash-recovery-authorized process)
- **Stale lock recovery:** If the file exists BUT the PID is dead AND no valid PREPARED
  marker matches the `activation_id`:
  - Log a stale-lock warning
  - Remove the lock file
  - Re-attempt acquisition with the new PID and activation_id
- **Pre-PREPARED crash recovery:** If the lock file exists, the PID is dead, and no PREPARED
  marker exists for the same activation_id:
  - This indicates a crash between lock acquisition and PREPARED marker creation
  - The lock is stale; apply stale lock recovery as above
  - The migration must re-start from scratch (no PREPARED state to resume from)
- **Lock release:** Remove the lock file after the COMMITTED marker is written (success) or
  after migration abort (cleanup). Lock release on abort MUST happen even if the process
  receives SIGTERM.

This design separates persistent maintenance intent (encoded in the PREPARED marker's
activation_id) from OS-level lock ownership (encoded in the PID field), so that a crash
does not leave an orphaned maintenance state.

#### 7b — Phase markers (PREPARED/COMMITTED/CLEANED)

Phase markers are written to `.factory/migration-state/{migration-id}-state.json`:

- **PREPARED:** all staging writes complete; source fingerprint recorded; all under-exclusion
  validation checks (§Decision 4c) pass. PREPARED contains:
  - `activation_id`: must match the manifest and the lock file
  - `source_sha256`: SHA-256 of the source file (BC-INDEX.md for B2) at census time
  - `completed_renames`: initially empty list; updated per §7c as each rename completes
  - `staging_paths`: map of target → staging path for each file being replaced

  BC-INDEX.md's original body is UNTOUCHED at this point. The original content is preserved
  in the staging copy; `staging_paths` records where each staging file lives.

- **COMMITTED:** written AFTER the last rename in the atomic-replace sequence (§7c).
  COMMITTED is the single commit-pointer for the multi-file atomic operation. Its presence
  means ALL target files have been atomically replaced as a group.

- **CLEANED:** staging temporary files removed; terminal success state.

#### 7c — Atomic publication and per-target completion tracking (F2 — resolves COMMITTED contradiction)

**Root cause of v1.1 contradiction:** v1.1 asserted both "COMMITTED written AFTER the last
rename" and "before COMMITTED, the original body is untouched." Once BC-INDEX.md (a migration
target) is renamed to its shard-redirect form, its content changes before COMMITTED exists.
The TOCTOU guard (re-reading source fingerprint) would then find a mismatch and abort — but
abort requires the original body to be untouched, which it no longer is. This is a genuine
cycle.

**v1.2 resolution:** The TOCTOU check is performed EXACTLY ONCE, before the first rename.
Per-target completion tracking in the PREPARED marker enables crash recovery without
re-checking the (now-renamed) source.

**Atomic-replace sequence:**

1. TOCTOU check (before first rename): Re-read source file, compute SHA-256. Compare against
   `source_sha256` in PREPARED marker. If they differ: ABORT. Delete PREPARED marker. Source
   is untouched (no renames have occurred). Migration requires re-activation.

2. For each target file in the migration:
   a. Execute `rename(staging_path, target_path)` — atomic on POSIX within the same
      filesystem (guaranteed since staging paths are in the same directory tree as targets)
   b. Execute `fsync` on the parent directory of `target_path` (mandatory dir-fsync per
      Pillai et al. OSDI'14; ensures the directory entry survives a system crash)
   c. Append the completed target's path and expected-hash to `completed_renames` in the
      PREPARED marker (atomic overwrite of the marker file)
   d. If the rename or dir-fsync fails: ABORT. Target files listed in `completed_renames`
      have been replaced; targets NOT in `completed_renames` are untouched. See crash
      recovery below.

3. Write the COMMITTED phase marker. COMMITTED is written AFTER the last rename + dir-fsync
   + PREPARED update.

**Crash recovery from partial rename sequence:**

On restart (PREPARED exists, COMMITTED does not):
1. Read `completed_renames` list from PREPARED
2. For each path in `completed_renames`: compute SHA-256 of the current file, verify it
   matches the expected hash. If mismatch: treat as ABORT (data integrity failure; do not
   proceed; require human intervention)
3. Resume renaming from the first target NOT in `completed_renames`, using the corresponding
   staging path from `staging_paths`
4. Continue the atomic-replace sequence from step 2b above

This recovery does NOT require the source file to be untouched — the TOCTOU check already
passed before the first rename, and the staging files preserved in `staging_paths` provide
the correct content for any incomplete rename.

**Reader integration (BC-1.18.010 — addresses F2 reader gap):**

During the migration window (after first rename, before CLEANED), readers that access
BC-INDEX paths MUST consult the COMMITTED marker to determine the correct read path:
- COMMITTED present: the migration is complete; read from canonical shard paths
- COMMITTED absent: migration in progress or not yet started; read from legacy BC-INDEX.md

In steady state (after CLEANED), shard paths are always canonical; the COMMITTED marker is
archived to `.factory/migration-audit/`. The product-owner must amend BC-1.18.010 §Reader
Integration to specify this transient-to-steady-state transition (see §Downstream to
Product-Owner).

### Decision 8 — Policy exception declaration (v1.2 REDESIGNED — addresses F6 and F8)

This ADR declares a **narrowly authorized exception** to CLAUDE.md's TD-FACTORY-HOOK-BYPASS-001
governing rule ("Use Edit/Write tools ONLY for `.factory/` mutations").

The migration binary writes to `.factory/` paths outside the Edit/Write tool surface. This IS
a bypass of the standard hook-validated write path in the mechanical sense; it is authorized
only because POLICY 22 human ratification establishes that the migration binary's own internal
controls are equivalent to or stronger than the skipped controls.

#### Accurate skipped-control inventory (F6 — corrected from v1.1)

| Skipped control | What it actually enforces | Equivalent enforcement in migration binary or ratified waiver |
|-----------------|---------------------------|---------------------------------------------------------------|
| `brownfield-discipline` PreToolUse (Edit\|Write\|MultiEdit) | **Protects `.reference/` from governed writes** (script lines 4-9 and 34-38 check write targets against the `.reference/` path prefix) | Migration binary does not write to `.reference/`; all targets are explicitly enumerated in §Decision 8 allowed-paths list. **No `.reference/` writes possible by construction.** |
| `factory-branch-guard` PreToolUse (Edit\|Write\|MultiEdit) | **Checks that the active worktree is NOT the factory-artifacts branch before permitting a mutation** (script lines 67-82 verify worktree identity and branch name) | Migration binary operates on the main working tree, not the factory-artifacts worktree. The migration targets (shard files, BC-INDEX.md) are in the main worktree. This guard does not apply to the migration binary's execution context. **EXPLICIT WAIVER (POLICY 22 authorized):** migration is always run in the main worktree; the guard's branch-check intent is satisfied because main-worktree BC mutations are exactly what this migration is authorized to perform. |
| `validate-factory-path-staging` PreToolUse (Bash) | Validates `.factory/` writes via Bash use proper channels | Guard amended (§Decision 5b) with full-command pre-shell classifier and manifest content validation (§5c); the amendment IS the equivalent enforcement |
| `validate-factory-path-staged` PostToolUse (Bash) | **IS registered for PostToolUse Bash** (hooks-registry.toml entry for PostToolUse Bash); verifies `.factory/` mutations from Bash are properly staged for factory-artifacts | **MUST be amended** by devops-engineer to recognize the governed migration subcommands (same absolute-path-pinned forms as §Decision 3) and pass them through without requiring Edit/Write staging semantics. Post-migration, devops-engineer performs the factory-artifacts commit manually as the audit record (§Decision 6). Guard amendment is a cluster-5 activation deliverable. |
| `validate-count-propagation.sh` PostToolUse behavior | Verifies count propagation across BC-INDEX | Migration binary's independent census cross-check against `total_bcs` oracle provides equivalent verification before any write; count propagation is a post-migration steady-state concern |
| POL-3 "NEVER use Python/sed/echo bypass" | Prevents unstructured, unvalidated `.factory/` writes | **EXPLICIT WAIVER (POLICY 22 authorized):** The migration binary is VSDD-authored, TDD-covered (VP-132/VP-133/VP-134), adversarially-reviewed. It implements the full BC-1.18.008/BC-1.18.011 specification. This waiver is narrowly scoped to the two governed migration subcommands and expires when the migration completes. |

#### Allowed write targets for the POLICY 22 exception (F8 — expanded from v1.1)

The exception covers writes to the following paths ONLY:
- `.factory/specs/behavioral-contracts/BC-INDEX.md` (B2: replaced with shard-redirect form)
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-NN.md` (B2: new shard files)
- `.factory/cycles/*/` (A: the four config-specified append-log target files in active cycle
  directories — exact paths are config-driven per §Decision 3, not hardcoded)
- `.factory/migration-state/` (both migrations: phase markers, exclusive.lock, state JSON)
- `.factory/activation/` (manifest file: written by state-manager, read+consumed by binary)
- `.factory/migration-audit/` (audit record: written by state-manager after COMMITTED)

All other `.factory/` paths remain subject to the Edit/Write-only constraint without exception.

#### CLAUDE.md amendment text (F8 — for human application at ratification)

The human MUST apply the following amendment to `CLAUDE.md` as part of POLICY 22 ratification.
Do NOT apply this text now; it takes effect only when the human ratifies ADR-052.

**Location:** `## Conventions (Code-Level)` section, `### Forbidden patterns` table, the row
for the TD-FACTORY-HOOK-BYPASS-001 P0 governing rule ("Use Edit/Write tools ONLY for
`.factory/` mutations. NEVER use Python/sed/echo bypass. Enforced by POL-3.").

**Amendment:** Append the following block AFTER the existing governing rule row text:

```
ADR-052 v1.2 EXCEPTION (POLICY 22 ratified): The governed one-time shard migration binary
(`{project-root}/target/release/factory-dispatcher migrate-bc-index` and
`backfill-append-logs`) may write to the following paths ONLY:
  `.factory/specs/behavioral-contracts/BC-INDEX.md`
  `.factory/specs/behavioral-contracts/shards/`
  `.factory/cycles/*/` (config-specified append-log targets)
  `.factory/migration-state/`
  `.factory/activation/`
  `.factory/migration-audit/`

PRECONDITIONS (all must hold before any migration-target mutation):
(a) Invoked via Bash tool with one-time interactive human approval at F4 activation boundary
    (no standing allowlist entry in `.claude/settings.json`).
(b) Armed-activation manifest at `.factory/activation/` is present, unexpired, and validated
    under exclusion: repo root SHA, expected_total_bcs, three-way ARCH-INDEX parity check
    (loaded config == manifest == live), and activation_id correlation (ADR-052 §Decision 4).
(c) Migration binary invoked at absolute trusted path with executable digest verified against
    `.factory/activation/factory-dispatcher.sha256`, using the closed argument grammar from
    ADR-052 §Decision 3 — validated by full-command pre-shell classifier in guard per
    ADR-052 §Decision 5c.
(d) The 4 dispatcher guard amendments (`destructive-command-guard`,
    `validate-factory-path-staging`, `validate-heavy-op-delegation`,
    `validate-factory-path-staged` PostToolUse) have been deployed per ADR-052 §Decision 5b.
(e) Native admission gate in `executor.rs` is deployed, covering Edit/Write/MultiEdit/Bash
    (ADR-052 §Decision 5a).

POST-SUCCESS OBLIGATIONS (must be satisfied after COMMITTED is written; NOT simultaneous
preconditions):
(f) Durable factory-artifacts commit records the census stdout, consumed activation manifest,
    COMMITTED phase marker, and binary version as the NIST AU-9 tamper-evident audit record
    (ADR-052 §Decision 6). State-manager performs this commit; timing: after COMMITTED.
(g) State-manager archives the activation manifest to `.factory/migration-audit/`.

All other `.factory/` writes by agents remain subject to the Edit/Write-only constraint.
```

### Decision 9 — Resolving the S-25.06 Rule 7 contradiction (unchanged from v1.1)

S-25.06 Rule 7's original text ("agent-executed CLI invocation logged as an Edit/Write tool
call") is WITHDRAWN and replaced with:

> "The activation step (T-10 for mechanism A, the equivalent activation step for mechanism B2)
> is executed via the `Bash` tool with one-time interactive human approval at F4 activation
> (no standing settings.json allowlist). The migration binary is invoked at its absolute trusted
> path: `{project-root}/target/release/factory-dispatcher backfill-append-logs` (or
> `migrate-bc-index`). The invocation is NOT an Edit/Write tool call. It is a Bash execution of
> the native migration binary under ADR-052 §Decision 3's closed argument grammar, activated
> under ADR-052 §Decision 4's armed-activation manifest, producing a census report captured per
> ADR-052 §Decision 6."

Story-writer updates S-25.06 Rule 7 text accordingly in the next burst.

### Decision 10 — BC-1.18.010 Invariant 2: ARCH-INDEX mapping with three-way revision binding (v1.2 REDESIGNED — addresses F5)

The BC-S-prefix→SS-NN mapping used for first-level addressing is implemented as a
**config-embedded snapshot with a revision binding and three parity checks**:

- The mapping is embedded in the `[[shard]]` TOML config entry for the BC-INDEX artifact as a
  `subsystem_prefixes` table: `{ "BC-1" = "SS-01", "BC-2" = "SS-02", ..., "BC-10" = "SS-10" }`.
- The config entry MUST also contain `arch_index_sha`: the git SHA of the ARCH-INDEX.md commit
  from which the snapshot was generated. This binding is non-optional.

**Three-way activation-time parity check (F5 — closes the two-way gap in v1.1):**

At activation, under exclusion (after lock acquisition, as part of §Decision 4c validation),
the binary performs an explicit three-way equality check:

  `config.arch_index_sha` (loaded from the binary's embedded config)
  == `manifest.approved_arch_index_sha` (from the armed-activation manifest)
  == live ARCH-INDEX SHA (current HEAD commit SHA of ARCH-INDEX.md in factory-artifacts)

All three values must be equal. The migration fails CLOSED if any pair diverges.

This catches the case that v1.1 missed: a stale binary built against ARCH-INDEX revision A,
invoked with a fresh manifest specifying revision B. Both live ARCH-INDEX and the manifest
agree on revision B, but the binary's embedded config still has revision A. v1.1's two-way
check (live == manifest) would pass this case. The three-way check catches it:
`config.arch_index_sha (A) != manifest.approved_arch_index_sha (B)` — FAIL CLOSED.

A stale binary must be rebuilt against the current ARCH-INDEX revision before activation can
proceed.

**CI parity test** (`cargo test --test arch_index_parity`): greps ARCH-INDEX.md's Subsystem
Registry `BC-S Prefix` column at HEAD and diffs it against the config snapshot, failing on any
divergence. This test runs on every commit to `develop` and `main`. CI parity is
NECESSARY-BUT-NOT-SUFFICIENT; the three-way activation check is the sufficient condition.

**Stale-installed-config test**: a separate test (`cargo test --test stale_installed_config`)
verifies that the binary's embedded snapshot `arch_index_sha` matches the expected revision
used in the test fixture. This test detects stale binaries built against an older ARCH-INDEX.

`shard_manager.rs` reads the mapping from the deserialized config entry only, never from
ARCH-INDEX.md at runtime. This is consistent with BC-1.18.010 Invariant 2's prohibition on
independent hardcoding.

## Rationale

### Head-to-head mechanism evaluation

#### Option A — Hook-driven / dispatcher-internal explicitly-armed one-shot native action

**Viability: CONFIRMED.** v1.0's rejection of this option evaluated ONLY unconditional,
detection-based ("automatically-triggered") hook activation. That was a strawman. Corrected
evaluation:

- Anthropic docs (hooks page) explicitly state: "Hooks are shell commands that execute with
  your full user permissions." The WASM sandbox applies to hook plugins running inside the
  dispatcher's WASM executor, NOT to shell commands a hook executes via `exec_subprocess`.
- The dispatcher already executes native mutations as a PreToolUse side-effect via
  `shard_cap_precheck` (in `executor.rs`) whose fired verdict routes to
  `shard_manager::execute_roll` — a destructive seal-and-truncate-to-0 operation — as a
  PreToolUse side-effect. This proves the hook-internal native mutation pattern works
  end-to-end in this codebase.
- An explicitly-armed, one-shot hook that checks an armed-approval marker before mutating DOES
  satisfy the F4 human-intent gate — the marker itself is written at human direction at F4.
- Recovery (PREPARED/COMMITTED/CLEANED markers) is achievable.
- A read-only census subcommand exposes stdout evidence for D-449(a).

**Why Option A is not chosen (operational complexity, not viability):**

1. Requires a new `[[hooks]]` entry in `hooks-registry.toml` and a new WASM or bash-adapter
   hook plugin — additional infrastructure for a one-time operation.
2. The "explicitly-armed" check requires designing and implementing an armed-marker reader
   inside the hook plugin — more new code than Option B.
3. Census evidence path is less direct.
4. Option B achieves the same governance properties with lower implementation overhead.

**Option A is NOT rejected on viability grounds.** A future recurring migration use case
should reconsider Option A; its encapsulation within the dispatcher hook chain is
architecturally cleaner for ongoing operations.

#### Option B — One-time interactive Bash approval at F4 (CHOSEN)

- **Least privilege:** No standing permission in settings.json; the permission expires after
  the one-time interactive approval. Zero persistent attack surface after migration completes.
- **F4 gate:** Human sees the permission prompt AT the exact activation moment.
- **D-449(a):** Stdout capture is direct; no separate census subcommand needed.
- **Guard stack:** Requires the same guard amendments; no additional dispatcher infrastructure
  vs. Option A.
- **Audit trail:** Factory-artifacts commit provides the durable, tamper-evident record.

#### Option C — Hardened standing allowlist (absolute-path-pinned, closed grammar)

**Why Option C is not chosen:**

1. **Fails F3 (standing permission ≠ activation authorization):** A settings.json entry
   persists indefinitely after migration completes.
2. **NECESSARY-BUT-NOT-SUFFICIENT:** Only suppresses the Claude Code permission prompt; guard
   amendments still needed. Adds standing-permission surface with no governance benefit.
3. **Post-migration cleanup burden:** Entry must be actively removed after migration.

#### Rejected-alternatives summary

| Criterion | Option A (hook-driven) | Option B (one-time-interactive) [CHOSEN] | Option C (hardened allowlist) |
|-----------|----------------------|----------------------------------------|------------------------------|
| Least privilege | Best (no permission change) | Best (no standing entry) | Weakest (standing entry persists) |
| F4 human gate | Via armed-marker (indirect but valid) | Direct (permission prompt at activation) | Via allowlist install (F3 fails) |
| D-449(a) evidence | Via census subcommand | Direct stdout | Direct stdout |
| New infrastructure | New hook plugin + registry entry | None | None |
| Guard amendments | Same 4 as Option B (v1.2) | Same 4 | Same 4 |
| Viability | Confirmed (conflation corrected) | Confirmed | Conditional (security-viable if hardened) |
| Selected? | No (complexity) | **Yes** | No (F3 fails) |

### Why this is a narrowly-authorized exception, not "the opposite of a bypass"

v1.0's "OPPOSITE of a bypass" framing was incorrect. A bypass is defined by the MECHANISM
(writing to `.factory/` outside the Edit/Write hook surface), not by the quality of the
bypassing code. The migration binary writes to `.factory/` outside the Edit/Write surface —
that IS a bypass, mechanically. What makes it authorized is:
1. POLICY 22 human ratification
2. The binary's own internal controls are enumerated and verified as equivalent to the skipped
   controls (§Decision 8)
3. The exception is narrowly scoped (two subcommands, one activation, expires on completion)

## Consequences

### Positive

- Resolves the internally contradictory execution path in S-25.06 Rule 7.
- Option B provides the lowest standing-permission surface: zero persistent Bash permissions
  after migration completes.
- D-449(a) census-report-to-stdout satisfies the burst-log Dim-2 literal-shell-execution-
  evidence obligation with zero additional tooling.
- Factory-artifacts commit provides a tamper-evident audit record satisfying NIST AU-9.
- Three-way activation-time ARCH-INDEX parity check (§Decision 10) closes the stale-binary
  gap that v1.1's two-way check missed.
- Crash-atomicity phase markers with per-target completion tracking (§Decision 7) eliminate
  the COMMITTED-vs-"original untouched" contradiction in v1.1.
- Content-bearing lock file with PID+activation_id (§Decision 7a) enables owner-independent
  crash recovery, including pre-PREPARED crash paths.
- Explicit policy exception (§Decision 8) is honest, auditable, and bounded; skipped-control
  inventory is accurate.
- Full-command pre-shell classifier in guard (§Decision 5c) closes the "binary rejects
  metacharacters" logical impossibility — metacharacters are rejected before shell execution.

### Negative

- One-time interactive approval requires operator presence at F4 activation.
- Guard amendments (4 guards in v1.2 vs 3 in v1.1) add implementation work.
- Native admission gate in `executor.rs` (§Decision 5a) is an additional binary change beyond
  the guard amendments.
- The CLAUDE.md amendment must be applied by the human at ratification.
- `validate-factory-path-staged` PostToolUse Bash now requires explicit amendment (v1.1
  incorrectly claimed interactive approval bypassed this event).

### Neutral

- BC-1.18.011's crash-atomicity hardening (§Downstream to Product-Owner) is required before
  cluster-5 TDD implementation.
- S-25.06 Rule 7 text change is a correction of a false constraint, not a behavioral change.

## Downstream to Product-Owner (BC-1.18.011 and BC-1.18.010 hardening)

The architect specifies the following amendments; the product-owner writes all BC body changes.
Do NOT modify BC content directly; route to product-owner.

### BC-1.18.011 required amendments (ordered by precedence)

**Amendment 1 — Precondition 4 (Codex F6 / A/B2 scheduling coupling removal):**

Replace the current Precondition 4 text with:

> "The migration is independently gated on the F4 activation boundary. It has NO timing or
> ordering dependency on BC-1.18.008's mechanism-A backfill-split; the two migrations activate
> independently (each via its own armed-activation manifest per ADR-052 §Decision 4) and may
> run in any order. They share an F4 activation window by operational convenience, not by
> specification."

**Amendment 2 — Postcondition 6 (Codex F6 / A/B2 coupling in PC6):**

In the final sentence of Postcondition 6, replace "at the SAME F4 activation moment mechanism
A's own backfill (BC-1.18.008) runs" with "at F4 activation, as part of the same one-time B2
migration operation (independently of mechanism A's activation schedule)." The requirement that
SS-05/SS-06 sub-split occurs WITHIN the same B2 operation (not a separate follow-on) is
PRESERVED; only the A/B2 simultaneous-activation coupling is removed.

**Amendment 3 — Postcondition 7 scope clarification (Codex F6 / Invariant 4 attribution):**

Append to the end of Postcondition 7:

> "Note: this postcondition governs B2/Cohort-B independence only. A/B2 scheduling independence
> (that mechanism A and B2 activate independently at F4) is governed by Precondition 4 [as
> amended by ADR-052 §Decision 1]."

**Amendment 4 — New Precondition 5: Phase markers (research Q5 Gap 3 + 2nd Codex F2/F3):**

Insert after Precondition 4 (now amended):

> "5. A durable phase-marker file at `.factory/migration-state/migrate-bc-index-state.json`
>    is written at each phase transition and survives crashes:
>    - PREPARED: all staging writes complete, source fingerprint recorded, all under-exclusion
>      validation checks pass, `completed_renames` list initialized — BC-INDEX.md original body
>      untouched at this point (original content preserved in staging).
>    - COMMITTED: all atomic file replacements complete and per-target hashes verified; this
>      marker is the single commit-pointer for the multi-file atomic operation (ADR-052
>      §Decision 7). Written AFTER the last rename and dir-fsync.
>    - CLEANED: staging temporaries removed; terminal success state.
>    EC-003 resume logic reads the `completed_renames` field to determine which renames
>    succeeded and which must be retried; it does NOT re-run the full split from scratch."

**Amendment 5 — New Precondition 6: Writer exclusion / maintenance boundary (research Q5 Gap 5
+ 1st Codex F4 + 2nd Codex F1 — CORRECTED from v1.1):**

Insert after Amendment 4:

> "6. A WRITER-EXCLUSION maintenance boundary is in force during migration execution. The
>    migration binary acquires an exclusive advisory lock file at
>    `.factory/migration-state/exclusive.lock` (containing PID + activation_id + timestamp per
>    ADR-052 §Decision 7a) before any read of source files. ALL mutation tool calls
>    (Edit/Write/MultiEdit/Bash) targeting BC-INDEX paths are blocked by the native admission
>    gate in `executor.rs` (ADR-052 §Decision 5a) when this lock file exists with an alive PID.
>    Any write that completes during the pre-lock window is detected by the TOCTOU guard
>    (ADR-052 §Decision 7c) which aborts the migration and requires re-activation. The lock is
>    released only after the COMMITTED phase marker is written (success) or on migration abort."

Note: v1.1 Amendment 5 incorrectly said "Edit/Write tool calls validated by
`validate-factory-path-staging`". The correct specification is ALL mutation tool calls via the
native admission gate in `executor.rs`. The `validate-factory-path-staging` guard is an
additional layer for the Bash invocation path only.

**Amendment 6 — Postcondition 3: Dir-fsync mandate (research Q5 Gap 2):**

Append to the existing Postcondition 3 atomicity text:

> "Each atomic file replacement (rename(2) call) MUST be followed by fsync on the parent
> directory of the target file before proceeding to the next replacement, ensuring directory
> entries survive a system crash (POSIX rename(2) + dir-fsync semantics per Pillai et al.
> OSDI'14). The dir-fsync is mandatory, not best-effort."

**Amendment 7 — New Postcondition 3a: TOCTOU pre-commit guard (research Q5 Gap 4 + 2nd Codex F2):**

Insert immediately after Postcondition 3:

> "3a. Pre-commit source-fingerprint recheck (TOCTOU guard). This check is performed EXACTLY
>     ONCE, before the first rename in the atomic-replace sequence (not between individual
>     renames). The migration binary re-reads BC-INDEX.md's source content, computes SHA-256,
>     and compares against the `source_sha256` field recorded in the PREPARED marker. If they
>     differ: ABORT. The PREPARED marker is deleted. BC-INDEX.md's original body is left
>     untouched (no renames have occurred at this point). The migration requires re-activation.
>     This single-check design eliminates the v1.1 contradiction where a re-check after
>     BC-INDEX.md's own rename would find a fingerprint mismatch and incorrectly trigger abort."

**Amendment 8 — Invariant 3: commit-pointer specification (research Q5 Gap 1 + 2nd Codex F2):**

Append to the end of existing Invariant 3 text:

> "The all-or-nothing guarantee is implemented via the COMMITTED phase marker (Precondition 5
> [as amended]) and the `completed_renames` tracking in PREPARED: before COMMITTED exists, the
> state machine is either PREPARED (staging in progress, with zero or more completed renames
> tracked), or CLEAN (no migration in progress); after COMMITTED is written, the state is the
> split end-state. The COMMITTED marker is the sole commit-point for the multi-file atomic
> operation; composing N independent `write_atomic` calls without this commit-pointer does not
> satisfy this invariant."

**Amendment 9 — Architecture Anchors additions:**

Add the following anchors to the Architecture Anchors section:

> "- ADR-052 §Decision 4 — armed-activation manifest governing pre-mutation authorization
>    (two-phase validation: pre-lock and under-exclusion)
>  - ADR-052 §Decision 5a — native admission gate in executor.rs covering ALL mutation tools
>  - ADR-052 §Decision 7 — crash-atomicity phase markers + per-target completion tracking +
>    content-bearing lock file with crash-recovery ownership protocol
>  - ADR-052 §Decision 8 — POLICY 22 exception declaration with accurate skipped-control
>    inventory and enumerated allowed write targets"

### BC-1.18.010 required amendments (v1.2 additions)

**Invariant 2 amendment (1st Codex F7 + 2nd Codex F5 — three-way parity):**

Replace the current Invariant 2 text with:

> "The BC-S-prefix→SS-NN mapping used for first-level addressing is validated against
> ARCH-INDEX's Subsystem Registry at CI time and at migration activation time via an explicit
> three-way parity check, never independently hardcoded. At runtime, `shard_manager.rs` reads
> the mapping from the config entry only (a snapshot embedding the `arch_index_sha` of the
> ARCH-INDEX commit it was generated from, per ADR-052 §Decision 10). Three parity checks
> enforce that the snapshot never diverges from ARCH-INDEX:
> (a) CI test (`arch_index_parity`) diffs the snapshot against HEAD ARCH-INDEX on every commit;
> (b) at migration activation (under exclusion), the binary performs a three-way check:
>     `config.arch_index_sha` == `manifest.approved_arch_index_sha` == live ARCH-INDEX SHA;
>     fails CLOSED if any pair diverges — including a stale binary (config at revision A)
>     invoked with a current manifest (revision B);
> (c) stale-installed-config CI test verifies the binary's embedded snapshot `arch_index_sha`
>     matches the expected revision used in the test fixture.
> A future ARCH-INDEX subsystem renumbering requires regenerating the config snapshot from the
> new ARCH-INDEX commit and triggering a CI parity failure as the forcing function."

**Reader integration amendment (2nd Codex F2 — new, not in v1.1):**

Add a new §Reader Integration section to BC-1.18.010:

> "During the B2 migration window (after the first target rename, before CLEANED), readers
> accessing BC-INDEX paths MUST consult the COMMITTED marker at
> `.factory/migration-state/migrate-bc-index-state.json` to determine the correct read path:
> - COMMITTED present: read from canonical shard paths (migration complete)
> - COMMITTED absent: read from legacy BC-INDEX.md path (migration in progress or not started)
> In steady state (after CLEANED), shard paths are always canonical. COMMITTED is archived to
> `.factory/migration-audit/` at CLEANED time; its absence in steady state is not an error."

### error-taxonomy.md correction (2nd Codex F1 — new, not in v1.1)

`error-taxonomy.md` line 84 incorrectly claims the maintenance lock is enforced on
Edit/Write tool calls by `validate-factory-path-staging`. The correct specification is:

> "The maintenance lock (E-MAINTENANCE) is enforced by the native admission gate in
> `executor.rs` (ADR-052 §Decision 5a), which fires BEFORE any registry plugin for ALL
> mutation tool calls: Edit, Write, MultiEdit, and Bash. The `validate-factory-path-staging`
> guard enforces the full-command classifier (ADR-052 §Decision 5c) for the Bash path only
> and is a secondary guard layer, not the primary maintenance-admission mechanism."

The product-owner or technical-writer must apply this correction to `error-taxonomy.md`
in the same burst as the BC hardening.

## References

- `adv-cv-adr052-v11-closure-2026-09-12.md` — 2nd Codex closure review, RATIFY-WITH-CHANGES
  8 findings (D-1216) that prompted this v1.2 redesign
- `adv-cv-adr052-cluster5-F1-2026-09-12.md` — 1st Codex RATIFY-WITH-CHANGES verdict (7
  findings, D-1214) that prompted v1.1 revision
- `research-adr-052-assumption-validation-2026-09-12.md` — research-agent Q1-Q5 validation
  (converged with Codex on DO-NOT-RATIFY)
- `adv-cv-dir-cluster5-F1-direction-2026-09-12.md` — CV-DIR-F2 finding that prompted v1.0
- `s2502-cluster5-f1-delta-analysis.md` §4 — original options evaluation (v1.0 basis)
- `BC-1.18.011` — migration BC; §Downstream to Product-Owner specifies amendments
- `BC-1.18.010` Invariant 2 + new §Reader Integration — amended per §Downstream to Product-Owner
- `S-25.06-append-log-backfill-split-executor.md` Rule 7 — corrected per §Decision 9
- `ADR-051` §Decision 1 (WASM fuel-budget constraint), §Decision 7 (B2 end-state),
  §Decision 10 (governed one-time B2 migration)
- `CLAUDE.md` §WASM plugin fuel budgets, TD-FACTORY-HOOK-BYPASS-001 P0 governing rule
  (see §Decision 8 for amendment text); human-only edit target — not modified by this ADR
- `executor.rs` `shard_cap_precheck` → `shard_manager::execute_roll` — existing native
  PreToolUse mutation pattern that proves Option A viability; native admission gate added here
- `plugins/vsdd-factory/hooks-registry.toml` — 13 `^Bash$` hook entries (9 PreToolUse);
  §Decision 5b enumerates required amendments (4 guards in v1.2)
- NIST SP 800-53r5 AU-9 (audit trail tamper-evidence requirement)
- Pillai et al. OSDI'14 "All File Systems Are Not Created Equal" (dir-fsync requirement)
- CWE-88 (argument injection), CWE-78 (OS command injection), CWE-22 (path traversal)

## Files to Change

| File | Change | Owner |
|------|--------|-------|
| `CLAUDE.md` | Apply §Decision 8 CLAUDE.md amendment text to the TD-FACTORY-HOOK-BYPASS-001 P0 governing rule in the Forbidden patterns table | **Human only** (human-mandated-edit-only; NOT agent-editable) |
| `.claude/settings.json` | No change — no Bash allowlist entry added (Option B requires no settings.json mutation) | — |
| `.factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md` | Apply §Downstream to Product-Owner Amendments 1-9 | product-owner |
| `.factory/specs/behavioral-contracts/ss-01/BC-1.18.010.md` | Apply §Downstream to Product-Owner Invariant 2 amendment + new §Reader Integration section | product-owner |
| `error-taxonomy.md` | Correct line 84 claim per §Downstream error-taxonomy.md correction | product-owner or technical-writer |
| `plugins/vsdd-factory/hooks-registry.toml` | Amend `destructive-command-guard`, `validate-factory-path-staging`, `validate-heavy-op-delegation`, `validate-factory-path-staged` (PostToolUse Bash) per §Decision 5b; add writer-exclusion lock-check and full-command classifier to `validate-factory-path-staging` per §Decision 5c | devops-engineer (cluster-5 activation) |
| `plugins/vsdd-factory/hooks/destructive-command-guard.sh` (or equivalent WASM) | Implement full-command pre-shell classifier (§Decision 5c); exception for absolute-path-pinned `factory-dispatcher migrate-bc-index` and `backfill-append-logs` after classifier pass | devops-engineer |
| `plugins/vsdd-factory/hooks/validate-factory-path-staging.sh` (or equivalent WASM) | Implement full-command pre-shell classifier (§Decision 5c) including executable digest verification and manifest content validation (not presence-only); add writer-exclusion lock-presence check | devops-engineer |
| `plugins/vsdd-factory/hooks/validate-factory-path-staged.sh` (or equivalent WASM) | Amend PostToolUse Bash handler to recognize governed migration subcommands and pass them through | devops-engineer |
| `crates/factory-dispatcher/src/executor.rs` | Add native admission gate check (§Decision 5a) at top of dispatch path, before `shard_cap_precheck`, covering Edit/Write/MultiEdit/Bash; check for `exclusive.lock` and alive PID | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/src/shard_manager.rs` | Add `run_mechanism_b2_bc_index_split` entry point; add `migrate-bc-index`/`backfill-append-logs` CLI subcommands + census stdout; implement phase markers with `completed_renames` tracking (§Decision 7b/7c); implement content-bearing lock file with PID+activation_id (§Decision 7a); implement single TOCTOU check before first rename (§Decision 7c); implement dir-fsync per rename; implement two-phase armed-activation manifest validation (§Decision 4b/4c); implement three-way ARCH-INDEX parity check (§Decision 10); implement manifest consumption (add consumed_at) | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/tests/` | Add `arch_index_parity` CI test; add `stale_installed_config` test (§Decision 10); add two-phase manifest validation tests; add per-target completion tracking + crash-recovery tests; add TOCTOU single-check tests; add stale-lock recovery tests; add pre-PREPARED crash tests; add full-command classifier negative tests (§Decision 5c); add three-way parity failure tests (§Decision 10); add concurrent-write admission gate tests (§Decision 5a) | implementer (cluster-5 TDD) |
| `.factory/activation/factory-dispatcher.sha256` | SHA-256 of the built `factory-dispatcher` binary, written at cluster-5 activation package build time | devops-engineer (cluster-5 activation) |
| `.factory/activation/` (new directory) | Created at F4 by state-manager when writing the armed-activation manifest | state-manager (F4 activation burst) |
| `.factory/migration-state/` (new directory) | Created by migration binary at runtime for phase markers, `completed_renames`, lock file | migration binary (runtime) |
| `.factory/migration-audit/` (new directory under factory-artifacts worktree) | Created by state-manager when committing the durable audit record (§Decision 6) | state-manager (post-migration burst) |

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.2 | 2026-09-13 | architect | Full redesign per D-1216 (2nd Codex RATIFY-WITH-CHANGES 8 findings). Resolves: F1 — native admission gate added in `executor.rs` covering ALL mutation tools (Edit/Write/MultiEdit/Bash) before shard_cap_precheck, replacing the `^Bash$`-only guard-level check; drain protocol via TOCTOU abort. F2 — per-target completion tracking in PREPARED marker + single TOCTOU check before first rename (not repeated between renames) resolves COMMITTED-after-last-rename vs "original untouched" contradiction; reader integration protocol specified (COMMITTED marker as read-path selector for BC-1.18.010). F3 — content-bearing lock file with PID+activation_id separates persistent maintenance intent from OS advisory lock; pre-PREPARED crash recovery path defined; stale-lock recovery specified. F4 — two-phase manifest validation: pre-lock (lightweight) then under-exclusion (repo_root_sha, expected_total_bcs, three-way ARCH-INDEX parity, readiness); pre-publication expiry recheck before COMMITTED; manifest CONSUMED marking replaces deletion (resolves "absent=rejected" vs "already-migrated=exit0" contradiction); three expiry-safe recovery modes defined. F5 — explicit three-way activation-time parity check: config.arch_index_sha == manifest.approved_arch_index_sha == live ARCH-INDEX SHA; catches stale-binary case (revision A config + revision B manifest + revision B live). F6 — accurate skipped-control inventory: brownfield-discipline description corrected to ".reference/ write protection"; factory-branch-guard row added with explicit waiver; validate-factory-path-staged PostToolUse Bash corrected from "bypassed" to "MUST be amended." F7 — full-command pre-shell classifier moved to guard layer (validate-factory-path-staging + destructive-command-guard) with executable digest verification; removed impossible "binary rejects metacharacters post-shell-parse" claim; negative tests specified. F8 — CLAUDE.md amendment text expanded to include BC-INDEX.md + migration-state/ + activation/ + migration-audit/ + config-specified mech-A targets; preconditions (a-e) separated from post-success obligations (f-g); CLAUDE.md rule referenced by text anchor not line number; 4 guard amendments listed (vs 3 in v1.1). Downstream to Product-Owner: Amendment 5 corrected (Edit/Write → ALL mutation tools via native admission gate); error-taxonomy.md correction added; BC-1.18.010 §Reader Integration section added. |
| 1.1 | 2026-09-12 | architect | Full revision per D-1214 (1st Codex RATIFY-WITH-CHANGES 7 findings + research Q1-Q5). Mechanism changed from Option C (hardened standing allowlist) to Option B (one-time interactive Bash approval at F4, no settings.json change). Option A re-evaluated honestly. Declares explicit narrowly-authorized policy exception (§Decision 8). Activation-manifest authorization mechanism added (§Decision 4). 9 PreToolUse `^Bash$` dispatcher guards enumerated (§Decision 5). Audit trail upgraded to factory-artifacts commit satisfying NIST AU-9 (§Decision 6). Crash-atomicity phase markers, TOCTOU pre-commit guard, dir-fsync mandate added (§Decision 7). Config snapshot bound to ARCH-INDEX revision with activation-time parity check (§Decision 10). BC-1.18.011 Amendments 1-9 and BC-1.18.010 Invariant 2 amendment specified. NOT RATIFIED per D-1216. |
| 1.0 | 2026-09-12 | architect | Initial authoring. Resolves CV-DIR-F2 (impossible execution path). Specifies Bash-tool-with-allowlist as sanctioned invocation path for both mechanism-A (S-25.06) and mechanism-B2 (BC-1.18.011) migrations. NOT RATIFIED per D-1214. |
