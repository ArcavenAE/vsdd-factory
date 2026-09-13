---
document_type: architecture-decision-record
level: L3
version: "1.3"
status: proposed
producer: architect
timestamp: 2026-09-13T00:00:00Z
phase: F1
subsystems_affected:
  - SS-01
traces_to: .factory/specs/architecture/ARCH-INDEX.md
inputs:
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-adr052-v12-closure-2026-09-13.md
  - .factory/cycles/v1.0-brownfield-backfill/research-adr-052-v13-atomic-publication-2026-09-13.md
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-adr052-v11-closure-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-dir-cluster5-F1-direction-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/adv-cv-adr052-cluster5-F1-2026-09-12.md
  - .factory/cycles/v1.0-brownfield-backfill/research-adr-052-assumption-validation-2026-09-12.md
  - .factory/stories/S-25.06-append-log-backfill-split-executor.md
  - .factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md
  - .factory/cycles/v1.0-brownfield-backfill/s2502-cluster5-f1-delta-analysis.md
  - CLAUDE.md
  - .claude/settings.json
  - crates/factory-dispatcher/src/main.rs
  - plugins/vsdd-factory/hooks-registry.toml
input-hash: "0f2eada"
# input-hash: run compute-input-hash --update at state-manager registration burst
---

# ADR-052: Native Migration CLI — Sanctioned Execution Path for Governed One-Time Shard Migrations (v1.3 Redesign)

## Status

PROPOSED — v1.2 NOT ratified per D-1218 (3rd Codex cross-vendor closure review, 11 findings,
novelty HIGH, trajectory DIVERGING 7→8→11). This v1.3 redesign adopts the atomic-pointer
architecture recommended by `research-adr-052-v13-atomic-publication-2026-09-13.md` and
resolves all 11 findings. Human POLICY 22 ratification required before this ADR is treated
as accepted. Cluster-5 TDD dispatch remains BLOCKED until ratification.

## Context

Two governed one-time shard migrations are specified in S-25.02:

1. **Mechanism-A append-log backfill-split** (BC-1.18.008) — wired and executed by S-25.06.
   `run_mechanism_a_backfill_split` exists in `shard_manager.rs` with zero production callers.
   S-25.06 adds the CLI entry point and executes the migration against the four live cycle
   append-log files.

2. **Mechanism-B2 BC-INDEX body-split migration** (BC-1.18.011) — cluster-5 of S-25.02.
   The `run_mechanism_b2_bc_index_split` function (to be authored in cluster-5 TDD) is the
   native migration entry point in `shard_manager.rs`.

**Core context (unchanged from v1.2):** Both migrations share a structural challenge requiring
a narrowly-authorized native binary exception to POL-3 / TD-FACTORY-HOOK-BYPASS-001 P0. The
constraints (native binary needed; hook-validated Edit/Write path insufficient for multi-MB
files), the Option A/B/C evaluation, and the selection of Option B (one-time interactive Bash
approval) are unchanged. This ADR now additionally adopts a crash-safe atomic publication model
recommended by research to resolve the 3rd Codex review findings.

**v1.3 root-cause correction:** The 3rd Codex review correctly identified that publishing a
multi-file migration by renaming each target file in place through an ordered sequence cannot
provide all-or-nothing multi-file visibility regardless of journaling. Readers observing any
intermediate state see a partially-committed generation. v1.3 replaces this model with a single
atomic pointer swap over an immutable staging generation, a framed checksummed intent log for
per-target recovery, an advisory flock on a stable never-unlinked inode, and a real drain
(OPEN/DRAINING gate with writer reservations spanning PreToolUse→tool-completion).

**Correction from v1.2 (unchanged):** `.claude/settings.json` contains ONLY `enabledPlugins`;
no existing Bash permission entries. Dispatcher constraint, Option A viability, and native
admission gate context are unchanged from v1.2 §Context.

## Finding → Resolution (v1.3)

All 11 findings from the 3rd Codex cross-vendor closure review (`adv-cv-adr052-v12-closure-2026-09-13.md`) are resolved by this redesign:

| Finding | Sev | Root cause in v1.2 | Resolution in v1.3 |
|---------|-----|-------------------|-------------------|
| F1 | HIGH | Per-file rename: reader between renames sees mixed generation; "COMMITTED absent → read legacy" heuristic is ambiguous after crash | §Decision 7c: single atomic CURRENT.json pointer swap; reader resolves pointer once, pins one generation; COMPLETED.json as permanent terminal record eliminates ambiguous-absence heuristic |
| F2 | HIGH | Crash after rename but before `completed_renames` append: staging gone, no record, retry cannot recover | §Decision 7b: framed checksummed intent log records per-target expected post-content hash + expected pre-state BEFORE first rename; recovery accepts matching destination hash as completion; fail closed on ambiguity |
| F3 | HIGH | "Drain protocol" only detects writes already completed; admits writers before lock; nothing waits for in-flight writer to finish | §Decision 5a: real OPEN/DRAINING gate; writer reservations span PreToolUse→tool-completion; coordinator waits for active_writer_count=0 before snapshotting; fingerprint recheck kept as defense-in-depth |
| F4 | HIGH | O_CREAT\|O_EXCL + unlink-reclaim: two reclaimers both classify inode as stale, one unlinks the other's live inode; empty/corrupt lock classified as "dead → unlink" | §Decision 7a: advisory flock(LOCK_EX) on stable pre-created never-unlinked inode; stale reclamation automatic on process death; fail closed on empty/corrupt metadata by acquire-first (never unlink) |
| F5 | HIGH | No exclusive takeover path for dead-PID PREPARED transaction; BC-1.18.011 and error-taxonomy.md block writers only for alive PID, contradicting "block for any PREPARED" | §Decision 7a: durable txn record separate from flock; ordinary writers blocked by txn record state (PREPARED or COMMITTING) regardless of PID liveness; recovery-owner bumps fencing generation to claim ownership; releasing flock never deletes txn record |
| F6 | HIGH | Expiry recheck placed after all renames (§4d in v1.2); expiry can delete PREPARED after canonical content already changed; no authorized path for new activation to recover old transaction | §Decision 4d: authorization gate moved to immediately before CURRENT.json pointer swap (single pivot); post-pivot COMMITTING state retained regardless of expiry; completion-only recovery manifest bound to old activation_id + staged-generation hash + monotonic fencing generation |
| F7 | HIGH | Admission gate conditions on "target path under protected directory" for Bash, but Bash commands yield no intrinsic target paths; unknown write effects undefined | §Decision 5c: conservative Bash admission block: while txn record exists in PREPARED/COMMITTING state, block all Bash commands with unknown write effects; classify only sanctioned migration commands from config-driven target sets; canonicalization and alias rejection specified |
| F8 | MED | Guard requires unexpired manifest for --census (no manifest needed) and COMMITTED reruns (terminal no-op needs no manifest); both paths blocked after manifest archival | §Decision 5c: four separate guard branches: read-only census, validated terminal-state no-op, new activation, and recovery; manifest requirement and flock acquisition apply only to new-activation and recovery branches |
| F9 | HIGH | After CLEANED archives COMMITTED, absent COMMITTED is indistinguishable between "never ran" and "completed long ago"; reader and rerun logic cannot determine steady state | §Decision 7c: COMPLETED.json written at stable path after all target moves; permanent, never deleted, never archived; supersedes CURRENT.json for reader protocol; closes steady-state ambiguity |
| F10 | HIGH | Allowed-paths list in §Decision 8 omits sub-shards (.a.md/.b.md), BC-INDEX.shard-manifest.toml, BC-INDEX-SS-05.manifest.toml; CLAUDE.md amendment uses "shards/" directory which is broader scope | §Decision 8: single authoritative allowlist enumerating all required migration output paths including sub-shards and manifest files; CLAUDE.md amendment generated from that exact same list with containment checks |
| F11 | HIGH | Guard hashes binary at path, then shell executes same path by name; concurrent cargo build can replace binary between hash check and exec; TOCTOU window implicit | §Decision 11: Linux: open fd, hash through fd, execveat(fd, "", argv, envp, AT_EMPTY_PATH) — eliminates TOCTOU; macOS: fexecve unavailable per Apple documentation; decision: freeze build under maintenance lock + document residual window; requires human sign-off (see §Decision 11) |

---

## Decision

### Decision 1 — Mechanism selection: one-time interactive Bash approval (Option B) (unchanged)

Governed one-time shard migrations are invoked via the **`Bash` tool with one-time interactive
human approval at F4 activation time — NO standing allowlist entry in settings.json.**

The agent invokes `{project-root}/target/release/factory-dispatcher migrate-bc-index` at the
F4 activation step. The Claude Code harness shows a permission prompt; the human approves once.
The migration runs, completes, and the permission expires. This is Option B. See §Rationale for
the head-to-head evaluation of Options A, B, C (unchanged from v1.2).

### Decision 2 — Binary placement (unchanged from v1.0)

Both migration entry points live in the `factory-dispatcher` binary
(`crates/factory-dispatcher/`), co-located with `shard_manager.rs`. Rationale is unchanged:
co-location with `run_mechanism_a_backfill_split`, consistency with existing CLI dispatch layer.

### Decision 3 — Invocation: absolute-path-pinned, closed argument grammar (unchanged)

The agent MUST invoke the migration binary at its **absolute trusted path**:

```
{project-root}/target/release/factory-dispatcher migrate-bc-index
{project-root}/target/release/factory-dispatcher backfill-append-logs
{project-root}/target/release/factory-dispatcher migrate-bc-index --census
{project-root}/target/release/factory-dispatcher backfill-append-logs --census
```

The accepted argument grammar is CLOSED: exactly the above four forms. No additional flags,
no path arguments, no shell metacharacters, no compound commands. Any deviation is rejected by
the guard's pre-shell classifier (§Decision 5c) BEFORE shell execution, and rejected again by
the binary with a non-zero exit if somehow bypassed. No entry is added to `.claude/settings.json`.

### Decision 4 — Activation authorization (v1.3 REDESIGNED — closes F6)

#### 4a — Armed-activation manifest structure

State-manager writes the manifest to `.factory/activation/migrate-bc-index-YYYY-MM-DD.json`
(or `backfill-append-logs-YYYY-MM-DD.json`) at the F4 human-directed activation step.

The manifest MUST contain:
- `activation_id`: unique UUID for this activation
- `migration_id`: `"migrate-bc-index"` or `"backfill-append-logs"`
- `repo_root_sha`: factory-artifacts HEAD SHA at approval time
- `approved_arch_index_sha`: ARCH-INDEX committed SHA at approval time (§Decision 10)
- `expected_total_bcs`: integer from BC-INDEX frontmatter at approval time (B2 only)
- `approved_by`: `"human-F4-interactive"`
- `timestamp_utc`: ISO-8601 manifest creation timestamp
- `expires_after_hours`: `24`

**v1.3 addition:** `staged_generation_id` is NOT pre-specified in the manifest; it is filled
in by the migration binary when it creates the staging generation (see §Decision 7c step 1).
The manifest is then amended with `staged_generation_id` before the authorization gate check.

#### 4b — Pre-lock manifest checks (lightweight, before flock acquisition)

Before acquiring the advisory flock (§Decision 7a), the binary performs only:
1. Manifest file exists at the expected path — fails CLOSED if absent
2. `migration_id` matches the running subcommand — fails CLOSED on mismatch
3. Manifest is parseable — fails CLOSED on parse failure

These prevent flock acquisition against a clearly invalid activation state.

#### 4c — Under-exclusion validation (after flock acquisition, before staging generation build)

After acquiring the advisory flock (§Decision 7a), before any filesystem mutation:
1. Re-verify manifest exists and `migration_id` still matches
2. Current factory-artifacts HEAD SHA matches `repo_root_sha` — fails CLOSED on mismatch
3. `expected_total_bcs` matches current BC-INDEX frontmatter `total_bcs` — fails CLOSED on mismatch
4. Three-way ARCH-INDEX parity check (§Decision 10): `config.arch_index_sha` ==
   `manifest.approved_arch_index_sha` == live ARCH-INDEX SHA — fails CLOSED if any pair diverges
5. All readiness preconditions per BC-1.18.011 §Preconditions pass

**Note:** The expiry check (`expires_after_hours`) is NOT performed here. It is performed
immediately before the single destructive step (§Decision 4d). This keeps the under-exclusion
validation lightweight while positioning the authorization gate at the correct architectural point.

#### 4d — Authorization gate: immediately before CURRENT.json pointer swap (v1.3 — closes F6)

**The expiry check MUST occur immediately before the CURRENT.json pointer swap (§Decision 7c
step 4) — the single pivot/commit point.** This is the only authorization gate location that
is both (a) after all staging work is complete (so a timeout during staging does not abort
recoverable work) and (b) before any irreversible canonical-path visibility change.

**Authorization check logic:**
- If current UTC time is within `expires_after_hours` of `timestamp_utc`: pass; proceed to
  pointer swap.
- If expired AND txn record shows state=STAGING (pre-pivot, no canonical paths changed):
  ABORT cleanly — delete staging generation, delete txn record, release flock. The activation
  window elapsed before the pivot was reached; no recovery record needed.
- If expired AND txn record shows state=COMMITTING (post-pivot already): this branch is only
  reached via the recovery path (§Decision 4e recovery case), which uses a separate
  completion-only authorization (see below). Ordinary expiry check is inapplicable here.

**Post-pivot invariant (NEVER violate this):** After the CURRENT.json pointer swap transitions
the txn record to state=COMMITTING, the transaction recovery record (txn record + intent log)
MUST be retained regardless of original manifest expiry. The COMMITTING state represents an
irreversible publication commitment: canonical readers are now directed to use staging-generation
paths. The authorization window governs ENTRY to the irreversible phase, not COMPLETION of it.

**Completion-only recovery re-authorization (replaces fresh activation for F6):**
When a recovery process encounters a COMMITTING transaction whose original manifest has expired,
it MUST use a **completion-only recovery manifest** to proceed. This is a distinct manifest type:
- Bound to the OLD `activation_id` (references, does not reuse)
- Bound to the staged-generation hash (computed from CURRENT.json `staged_generation_id`)
- Bound to `allowed_steps`: the set of remaining forward-recovery steps only (no pre-pivot steps)
- Contains the current `fencing_generation` from the txn record
- Approved by the human via a separate F4-level approval labelled `"human-F4-recovery"`

The completion-only manifest MUST be rejected by the binary if: `activation_id` does not match
the original, `staged_generation_id` does not match CURRENT.json, `allowed_steps` includes
pre-pivot steps, or the txn record `fencing_generation` does not match the manifest's value.

#### 4e — Manifest consumption and recovery modes

After migration reaches COMPLETED (COMPLETED.json written), the binary adds `consumed_at` to
the manifest (atomic overwrite). State-manager archives manifest to `.factory/migration-audit/`.

Re-run behavior is governed by CURRENT.json + COMPLETED.json state, NOT manifest presence:

| Stable state | Action |
|---|---|
| `COMPLETED.json` exists | Exit 0 with `ALREADY_MIGRATED`; no lock needed; no manifest needed |
| CURRENT.json `status: committing` + valid manifest | Recovery: acquire flock; validate completion-only manifest or unexpired original; complete forward recovery |
| CURRENT.json `status: committing` + manifest absent or expired | Abort with `RECOVERY_REQUIRES_REAUTHORIZATION`; human must issue completion-only manifest |
| CURRENT.json `status: staging` + valid original manifest | Resume from staging: validate under exclusion; re-run authorization gate; proceed to pointer swap |
| CURRENT.json absent or `status: none` + manifest absent | Reject as unauthorized; exit non-zero |
| `--census` flag | Read-only; no CURRENT.json, no manifest, no flock required |

### Decision 5 — Dispatcher PreToolUse guard stack + native admission gate (v1.3 REDESIGNED)

#### 5a — Native maintenance admission gate: OPEN/DRAINING with writer reservations (v1.3 — closes F3)

The admission gate in `executor.rs` MUST implement a real drain — not merely a fingerprint
recheck — by tracking in-flight admitted writer reservations.

**Gate states:**
- `OPEN`: normal operation; new writer reservations accepted
- `DRAINING`: maintenance intent declared; no new writer reservations; wait for
  `active_writer_count = 0`
- `LOCKED`: coordinator holds flock and has reached quiescence; no mutations admitted until
  gate returns to OPEN

**Writer reservation lifecycle:**
- **PreToolUse:** before admitting a mutation (Edit/Write/MultiEdit/Bash) targeting
  `.factory/specs/behavioral-contracts/` or `.factory/cycles/`:
  - If gate state = OPEN: atomically increment `active_writer_count`; proceed
  - If gate state = DRAINING or LOCKED: return E-MAINTENANCE immediately (do not increment)
- **PostToolUse:** after tool completion (success or failure): atomically decrement
  `active_writer_count`; signal waiters if `active_writer_count = 0`

**Drain procedure (executed by maintenance coordinator before building staging generation):**
1. Acquire advisory flock (§Decision 7a)
2. Atomically flip gate state `OPEN → DRAINING` (persisted to `.factory/migration-state/gate-state`)
3. Wait for `active_writer_count = 0` (quiescence); timeout at 30s → ABORT and release flock
4. Snapshot now-quiescent inputs (compute `source_sha256` of all source files)
5. Flip gate state `DRAINING → LOCKED` (gate stays LOCKED for entire migration window)
6. Write txn record with state=STAGING (§Decision 7a)

**PostToolUse release:** When migration completes (COMPLETED.json written), flip gate `LOCKED → OPEN`.

**Implementation note:** Within a single dispatcher process, `{gate_state, active_writer_count}`
is protected by one mutex + condvar. The mutex is released atomically while waiting on condvar
(`pthread_cond_wait` / `Condvar::wait` semantics). The gate state is also persisted to
`.factory/migration-state/gate-state` for crash-recovery visibility across processes.

**Fingerprint recheck:** The source fingerprint check (§Decision 7c step 1) is retained as
defense-in-depth after quiescence is reached. It is not the drain mechanism.

**Non-upgrade:** Do NOT use a shared→exclusive flock upgrade as the drain mechanism. POSIX
flock upgrade releases the shared lock before acquiring the exclusive lock; the transition is
not atomic and creates a window for concurrent writers to acquire shared locks. Use the explicit
OPEN→DRAINING→LOCKED state machine above.

#### 5b — Guard amendments for `^Bash$` PreToolUse guards (v1.3 — updated for new state machine)

The same 4 guards as v1.2 require amendment (updated to reference txn record state and
CURRENT.json rather than exclusive.lock PID check):

| Guard | Required action |
|---|---|
| `destructive-command-guard` | Amend to recognize absolute-path-pinned migration commands after §5c classifier pass; reference txn record state for maintenance check |
| `validate-factory-path-staging` | Full-command pre-shell classifier (§5c) + manifest content validation; txn record state check replaces exclusive.lock PID check |
| `validate-heavy-op-delegation` | Allow migration command when completion-only or ordinary armed-activation manifest is present and validated |
| `validate-factory-path-staged` (PostToolUse) | Recognize governed migration subcommands; pass through without Edit/Write staging semantics |

#### 5c — Full-command pre-shell classifier (v1.3 REDESIGNED — closes F7, F8)

The guard receives the full proposed Bash command string BEFORE shell execution. The classifier
operates in four distinct branches:

**Branch 1 — Read-only census (closes F8 census gap):**
- Trigger: command matches `{canonical-binary-path} {migrate-bc-index|backfill-append-logs} --census`
- No manifest required; no flock required; no CURRENT.json check required
- Executable digest verification still applies (§Decision 11)
- Pass immediately after digest check

**Branch 2 — Terminal-state no-op (closes F8 COMMITTED-rerun gap):**
- Trigger: `COMPLETED.json` exists at `.factory/migration-state/completed.json`
- No manifest required; no flock required
- Binary will exit 0 with `ALREADY_MIGRATED`; guard confirms this is the expected outcome
- Pass immediately

**Branch 3 — New activation:**
- Trigger: no `COMPLETED.json` + no CURRENT.json with `status: committing`
- Full validation: exact command string match; metacharacter rejection; alternate-path rejection;
  manifest content validation (parseable, `migration_id` matches, not expired, `activation_id`
  is valid UUID); executable digest verification (§Decision 11)
- If any check fails: REJECT with descriptive error

**Branch 4 — Recovery:**
- Trigger: CURRENT.json `status: committing` exists
- Full validation: command string match; metacharacter rejection; executable digest verification;
  completion-only manifest validation (§Decision 4d) OR unexpired original manifest
- If any check fails: REJECT

**Conservative Bash admission for all other commands (closes F7):**
When the txn record exists with state=PREPARED or COMMITTING, ANY Bash command not matching
one of the four exact sanctioned forms (§Decision 3) MUST be BLOCKED if the command has any
write effect on paths overlapping the allowed-paths list (§Decision 8). "Unknown write effect"
means: shell metacharacters (`>`, `>>`, `|`, `&&`, `;`, `||`) OR commands not in an allowlist
of known read-only forms (ls, cat, grep, git log, git status, git diff, etc.). Unknown-effect
commands are fail-closed: if the guard cannot determine write effect is absent, BLOCK.

**Canonicalization and alias rejection:** The executable path is resolved via `realpath()` at
guard evaluation time. A relative path, a symlink pointing outside the project tree, or a
PATH-resolved name (no path separator) is rejected at Branch 3/4 regardless of final resolution.
The canonical project root is resolved via `git rev-parse --show-toplevel` at guard load time.

**Negative tests (cluster-5 TDD scope, unchanged from v1.2 with addition):**
- `{root}/target/release/factory-dispatcher migrate-bc-index && rm -rf /tmp/test` → REJECTED
- `/tmp/evil/factory-dispatcher migrate-bc-index` → REJECTED
- `{root}/target/release/factory-dispatcher migrate-bc-index --extra-flag` → REJECTED
- `{root}/target/release/factory-dispatcher` (no subcommand) → REJECTED
- Any Bash command with `>` targeting `.factory/specs/behavioral-contracts/` while txn=COMMITTING → REJECTED (F7 test)
- `{root}/target/release/factory-dispatcher migrate-bc-index` (when COMPLETED.json exists) → BRANCH-2 pass

### Decision 6 — Audit trail: durable, tamper-evident record (NIST AU-9) (unchanged from v1.2)

(See v1.2 §Decision 6 — no change. Census-to-stdout as D-449(a) evidence; factory-artifacts
commit as the NIST AU-9 tamper-evident record. The COMPLETED.json path is added to the
factory-artifacts commit in addition to COMMITTED phase marker.)

### Decision 7 — Crash-atomicity, lock ownership, and atomic publication (v1.3 REDESIGNED)

#### 7a — Lock ownership: advisory flock on stable never-unlinked inode (closes F4, F5)

**Lock inode:** `.factory/migration-state/exclusive.lock` — created ONCE by devops-engineer
at cluster-5 activation preparation and NEVER deleted, NEVER unlinked. Its presence as a
stable inode is guaranteed before any migration activation.

**Flock acquisition:**
```
fd = open(".factory/migration-state/exclusive.lock", O_RDWR)
result = flock(fd, LOCK_EX | LOCK_NB)
```
- If `result == EWOULDBLOCK`: another process holds the lock → return E-MAINTENANCE immediately.
  Do NOT inspect, unlink, or attempt to reclaim based on lock file contents.
- If `result == 0`: no live cooperating owner holds the lock (kernel guarantees: lock released
  on all fd closes including process death). The new owner MAY now update the lock file's
  diagnostic JSON body.

**Lock file diagnostic body (written under the held flock, loop until complete):**
```json
{
  "pid": <integer>,
  "activation_id": "<UUID from manifest>",
  "fencing_generation": <integer>,
  "timestamp_utc": "<ISO-8601>"
}
```
This body is INFORMATIONAL ONLY. It is NOT the authorization source of truth. Fail-closed
behavior: if, under the held flock, the file is found empty or corrupt from a prior partial
write, the new owner TRUNCATES and rewrites the full body under the held lock. The body is
never used to authorize reclamation decisions — only the flock state (held/not-held) is
authoritative for lock ownership.

**Durable txn record (separate from lock file — closes F5):**
`.factory/migration-state/txn-<activation_uuid>.json` is the PERSISTENT maintenance intent
record. It contains:
- `txn_id`: UUID (same as `activation_id`)
- `activation_id`: from manifest
- `fencing_generation`: monotonic integer, starts at 1, incremented by each recovery-owner
- `state`: `STAGING` | `COMMITTING` | `COMPLETED` | `ABORTED`
- `generation_id`: UUID of the staging generation (`null` until assigned at §Decision 7c step 1)
- `source_sha256`: SHA-256 of all source files at quiescence snapshot
- `intent_log_path`: path to the framed intent log (§Decision 7b)
- `pending_canonical_moves`: list of `{staging_path, canonical_path}` not yet completed
- `created_at`, `updated_at`: ISO-8601 timestamps

**Recovery-owner claim protocol (closes F5 exclusive takeover gap):**
A recovery process that acquires the flock after detecting a stale owner (txn record exists
with state=STAGING or COMMITTING) MUST claim the transaction by:
1. Reading the current `fencing_generation` from the txn record
2. Incrementing it by 1 (bump = claim)
3. Writing the updated txn record with the new `fencing_generation` and its own PID in the
   lock file body — under the held flock
4. Every subsequent write step by the recovery owner includes the current `fencing_generation`
   to prove authority

**Ordinary-writer blocking (closes F5 regardless-of-PID-liveness gap):**
The native admission gate (§Decision 5a) blocks ALL mutation tool calls targeting
`.factory/specs/behavioral-contracts/` or `.factory/cycles/` whenever a txn record exists at
`.factory/migration-state/txn-*.json` with state IN (STAGING, COMMITTING). This check is
independent of flock state and PID liveness. A txn record in STAGING or COMMITTING state means
the migration has exclusive rights to those paths, even if the flock-holding process is
temporarily absent (crash recovery scenario). Writers receive E-MAINTENANCE.

**Lock release rule:** The flock is released by closing the fd (including on process death via
kernel). RELEASING THE FLOCK NEVER DELETES THE TXN RECORD. A COMMITTING txn record survives
the release and blocks ordinary writers until a recovery owner resolves it to COMPLETED or
ABORTED.

#### 7b — Intent log: framed, checksummed, per-target expected hash (v1.3 — closes F2)

The intent log is `.factory/migration-state/intent-<generation_uuid>.log` — a sequence of
framed, checksummed records. Records are written in append-only mode and validated on read.

**Record format:**
```
--- INTENT_LOG_RECORD v1 ---
txn_id: <UUID>
fencing_generation: <integer>
record_type: INTENT | DONE | ABORTED
target_canonical: <path>
staging_path: <generation-path>
expected_post_hash: <sha256-hex>
expected_pre_state: missing | <sha256-hex>
timestamp_utc: <ISO-8601>
record_checksum: <sha256 of all above fields concatenated>
--- END_RECORD ---
```

A **torn record** (truncated, checksum mismatch, missing END_RECORD marker) MUST be treated
as absent — never as a partial INTENT or DONE. Recovery reads from the last valid record.

**Durable ordering (WAL boundary):**
1. Write + sync all staging generation files (fsync per §Decision 7d)
2. Append INTENT record per target to intent log; `fsync(intent_log_fd)`
   **WAL boundary: after this fsync, every rename is recoverable.**
3. `rename(staging_path, canonical_path)` (same filesystem — guaranteed since staging and
   targets are both under `.factory/`)
4. `sync_dir(parent_dir_of_canonical_path)` per §Decision 7d platform branches
5. Append DONE record for target; `fsync(intent_log_fd)`

**Recovery decision table (all cases fail closed except the two explicitly marked safe):**

| Canonical path state | Staging path state | Intent log record | Safe decision |
|---|---|---|---|
| `hash(canonical) == expected_post_hash` | any | INTENT present | **Treat DONE; append DONE record** |
| `hash(canonical) == expected_post_hash` | any | DONE present | Already complete; skip |
| `hash(canonical) == expected_pre_state` | `hash(staging) == expected_post_hash` | INTENT present | **Redo rename; sync dir; append DONE** |
| `canonical` missing, `expected_pre_state = missing` | `hash(staging) == expected_post_hash` | INTENT present | **Redo rename; sync dir; append DONE** |
| `hash(canonical) != expected_post_hash`, no INTENT in log | any | absent | FAIL CLOSED — ambiguous state |
| `hash(canonical) != expected_post_hash` | staging missing | any | FAIL CLOSED — no recovery copy |
| torn/invalid intent log frame | any | any | FAIL CLOSED — no trustworthy authority |
| `hash(canonical)` matches neither expected_post_hash nor expected_pre_state | any | any | FAIL CLOSED |

**Fault injection test mandate:** Tests MUST inject faults between every step:
staging-sync → intent-record-write → rename → dir-sync → done-record-write. Each fault
injection point must verify that recovery converges to the correct terminal state. A second
recovery pass on the already-recovered state must produce identical results (idempotent
forward recovery). Test tooling: `dm-log-writes` (Linux) or equivalent crash-simulation
at the `fsync`/`rename` boundary in test doubles.

#### 7c — Atomic publication: single CURRENT.json pointer swap (v1.3 — closes F1, F9)

**Generation directory layout:**
- Staging generation: `.factory/migration-state/gen-<uuid>/` — contains all new target files
  under their relative paths mirroring canonical structure
- Example: `.factory/migration-state/gen-<uuid>/BC-INDEX.md`,
  `.factory/migration-state/gen-<uuid>/shards/BC-INDEX-SS-01.md`, etc.
- This directory is IMMUTABLE after all content is written and synced

**Pointer file:** `.factory/migration-state/CURRENT.json` — the authoritative generation pointer.
The pointer transition is the single atomic commit point for the entire multi-file migration.

**Full atomic publication sequence:**

1. **Assign generation UUID.** Generate `generation_id = UUIDv4()`. Create generation directory
   `.factory/migration-state/gen-<uuid>/`. Write all new target file content under this directory.
   Update manifest with `staged_generation_id` (atomic overwrite of manifest). Update txn record
   with `generation_id` and `intent_log_path`.

2. **Sync all staging files.** For each file in the staging generation: `sync_file(fd)` per
   §Decision 7d platform branches. Then `sync_dir(staging_gen_dir_fd)` per §Decision 7d.

3. **Write intent log.** For each (staging_path → canonical_path) target: append INTENT record
   with `expected_post_hash = sha256(staging_file)` and `expected_pre_state = sha256(canonical_file)
   if exists else missing`. After all INTENT records: `fsync(intent_log_fd)`.
   **This is the WAL boundary: after this fsync, all renames are recoverable.**

4. **Authorization gate check (§Decision 4d).** Verify expiry. If expired and state=STAGING:
   ABORT cleanly. If passes: proceed.

5. **Fingerprint recheck (defense-in-depth).** Re-read all source files, recompute SHA-256.
   Compare against `source_sha256` in txn record (captured at quiescence). If any differ:
   ABORT; delete staging generation; txn record → ABORTED. Release flock. Require re-activation.

6. **Single atomic pointer swap (the commit point).** Write
   `.factory/migration-state/CURRENT.tmp.json` with content:
   ```json
   {"generation_id": "<uuid>", "status": "committing", "txn_id": "<activation_id>"}
   ```
   `sync_file(CURRENT_tmp_fd)` per §Decision 7d. Then:
   `rename(".factory/migration-state/CURRENT.tmp.json",
           ".factory/migration-state/CURRENT.json")` (atomic on POSIX/APFS).
   `sync_dir(migration_state_dir_fd)` per §Decision 7d.
   Update txn record: state → COMMITTING. `fsync(txn_record_fd)`.
   **After this rename, the migration is in COMMITTING state. There is no turning back.
   The authorization window is now irrelevant to completion.**

7. **Execute canonical path moves (forward-recoverable via intent log).** For each
   `(staging_path → canonical_path)` target:
   a. `rename(staging_path, canonical_path)` (same filesystem guaranteed)
   b. `sync_dir(parent_dir_of_canonical_path)` per §Decision 7d
   c. Append DONE record to intent log; `fsync(intent_log_fd)`
   d. Update `pending_canonical_moves` in txn record; `fsync(txn_record_fd)`
   If rename or dir-sync fails: DO NOT abort. Record failure; halt further renames; enter
   recovery state. Forward recovery (§Decision 7b) will resume from first uncompleted move.

8. **Write COMPLETED record (terminal — closes F9).** After ALL canonical path moves complete
   and verified (each `hash(canonical_path) == expected_post_hash`):
   Write `.factory/migration-state/completed.json`:
   ```json
   {"generation_id": "<uuid>", "txn_id": "<activation_id>",
    "completed_at": "<ISO-8601>", "canonical_paths_count": <N>}
   ```
   `sync_file(completed_json_fd)`. Update txn record: state → COMPLETED.
   This file is **PERMANENT. It is NEVER deleted, NEVER archived.** It is the authoritative
   steady-state terminal record. Its presence alone allows any reader or rerun to determine
   that migration is complete without consulting any other file.

9. **Cleanup (optional post-COMPLETED housekeeping).** Remove staging generation directory
   (no longer needed; canonical paths are authoritative). This step is optional; failure to
   clean up does not affect correctness. Cleanup is NOT a migration-state transition.

**Reader protocol (v1.3 — replaces v1.2 "COMMITTED absent → read legacy" heuristic):**
Readers that access BC-INDEX paths DURING the migration window MUST:
1. Check `.factory/migration-state/completed.json` — if exists: canonical paths are current;
   use canonical paths.
2. Check `.factory/migration-state/CURRENT.json` — if `status: committing`: migration is in
   progress; use `.factory/migration-state/gen-<generation_id>/` paths for reads (generation
   staging paths), NOT canonical paths.
3. If neither file exists: migration not started; use BC-INDEX.md (legacy form).
This protocol is unambiguous in all states including after crash recovery and cleanup.

#### 7d — Platform durability barriers (v1.3 — corrects v1.2 Amendment 6)

**Linux (ext4/xfs):**
- File sync: `fsync(fd)` — persists data + inode metadata
- Directory sync: `fsync(dir_fd)` — persists directory entry; MANDATORY after each `rename()`
  per Pillai et al. OSDI'14 — ensures directory entry survives power loss

**macOS/APFS (darwin-arm64):**
- File sync: `fcntl(fd, F_FULLFSYNC)` — required for power-loss durability on APFS. Apple's
  `fsync(2)` explicitly states: "the drive itself may not physically write the data to the
  platters for quite some time." `F_FULLFSYNC` asks the drive to flush all buffered data to
  permanent storage. **This is the only Apple-documented durability lever for APFS.**
- Directory sync: `fsync(dir_fd)` — Apple's `fsync(2)` and `fcntl(2)` do NOT document that
  fsync on an APFS directory fd provides power-loss durability. This call is BEST-EFFORT on
  macOS: execute it for ordering semantics, but do NOT treat it as a power-loss durability
  guarantee. **Documented residual risk:** under a power-loss scenario on APFS, a rename may
  survive without the directory entry being durable. A darwin-arm64 empirical durability test
  is required before this ADR is considered fully validated on macOS (see §Files to Change).
- `F_BARRIERFSYNC` is NOT a substitute for `F_FULLFSYNC`: it provides ordering but returns
  before earlier data necessarily reaches permanent media.

**Implementation:**
```rust
#[cfg(target_os = "macos")]
fn sync_file_durable(fd: RawFd) -> io::Result<()> {
    // F_FULLFSYNC required; plain fsync is insufficient for APFS power-loss durability
    if unsafe { libc::fcntl(fd, libc::F_FULLFSYNC) } != 0 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

#[cfg(not(target_os = "macos"))]
fn sync_file_durable(fd: RawFd) -> io::Result<()> {
    if unsafe { libc::fsync(fd) } != 0 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

// Both platforms: call but treat as best-effort on macOS/APFS
fn sync_dir_best_effort(dir_fd: RawFd) -> io::Result<()> {
    let _ = unsafe { libc::fsync(dir_fd) };
    Ok(())
}

#[cfg(not(target_os = "macos"))]
fn sync_dir_durable(dir_fd: RawFd) -> io::Result<()> {
    // Linux: mandatory for rename durability per Pillai et al. OSDI'14
    if unsafe { libc::fsync(dir_fd) } != 0 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}
```

**v1.2 Amendment 6 correction:** v1.2 Amendment 6 to BC-1.18.011 mandated "dir-fsync is
mandatory, not best-effort." This was correct for Linux but incorrect for macOS/APFS. The
BC-1.18.011 Amendment 6 replacement text is specified in §Downstream to Product-Owner.

### Decision 8 — Policy exception: single authoritative allowlist (v1.3 REDESIGNED — closes F10)

This ADR declares a **narrowly authorized exception** to CLAUDE.md's TD-FACTORY-HOOK-BYPASS-001
governing rule ("Use Edit/Write tools ONLY for `.factory/` mutations").

#### Skipped-control inventory (unchanged from v1.2)

See v1.2 §Decision 8 skipped-control inventory table. Controls and their equivalents in the
migration binary are unchanged. `validate-factory-path-staged` PostToolUse Bash still requires
explicit amendment. The POL-3 waiver scope is the two governed migration subcommands.

#### Single authoritative allowed-write-targets list (v1.3 — closes F10)

The exception covers writes to the following paths ONLY. This list is the SINGLE source of
truth for both the binary's containment checks AND the CLAUDE.md amendment text below:

```
# B2 migration targets (BC-INDEX body-split)
.factory/specs/behavioral-contracts/BC-INDEX.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>-A.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>-B.md
.factory/specs/behavioral-contracts/shards/BC-INDEX.shard-manifest.toml
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.manifest.toml

# A migration targets (config-driven append-log files)
.factory/cycles/*/  (exact paths config-driven per §Decision 3; not hardcoded)

# Migration operational state (both migrations)
.factory/migration-state/
.factory/migration-state/gen-<uuid>/

# Activation and audit (both migrations)
.factory/activation/
.factory/migration-audit/
```

Where `<NN>` is a two-digit subsystem number and `<uuid>` is the runtime generation UUID.

**Containment check (required before every write):** Before any write to any target path, the
migration binary MUST verify the resolved canonical path prefix matches one of the above entries.
Any target that does not match MUST be rejected with a non-zero exit regardless of manifest state.
This check is implemented in a `validate_write_target(path: &Path) -> Result<(), MigrationError>`
function called by every path that produces a filesystem write.

#### CLAUDE.md amendment text (v1.3 — generated from the same allowlist above)

The human MUST apply the following amendment to `CLAUDE.md` as part of POLICY 22 ratification.

**Location:** `## Conventions (Code-Level)` section, `### Forbidden patterns` table, the row
for TD-FACTORY-HOOK-BYPASS-001 P0.

**Amendment (generated from the authoritative allowlist above — no scope expansion):**

```
ADR-052 v1.3 EXCEPTION (POLICY 22 ratified): The governed one-time shard migration binary
(`{project-root}/target/release/factory-dispatcher migrate-bc-index` and
`backfill-append-logs`) may write to the following paths ONLY:
  `.factory/specs/behavioral-contracts/BC-INDEX.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>-A.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>-B.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX.shard-manifest.toml`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.manifest.toml`
  `.factory/cycles/*/` (config-specified append-log targets)
  `.factory/migration-state/` (flock inode, txn record, intent log, CURRENT.json, COMPLETED.json,
                                gate-state, staging generation)
  `.factory/activation/`
  `.factory/migration-audit/`

PRECONDITIONS (all must hold before any canonical-path mutation):
(a) Invoked via Bash tool with one-time interactive human approval at F4 activation boundary.
(b) Armed-activation manifest present, validated under exclusion: repo root SHA, expected_total_bcs,
    three-way ARCH-INDEX parity (config == manifest == live), activation_id correlation (ADR-052
    §Decision 4); OR completion-only recovery manifest per ADR-052 §Decision 4d.
(c) Migration binary invoked at absolute trusted path with executable digest verified via
    fd-binding on Linux (execveat AT_EMPTY_PATH) or freeze-build protocol on macOS
    (ADR-052 §Decision 11); closed argument grammar per ADR-052 §Decision 3.
(d) All 4 dispatcher guard amendments deployed per ADR-052 §Decision 5b.
(e) Native OPEN/DRAINING admission gate in `executor.rs` deployed (ADR-052 §Decision 5a);
    txn record state (STAGING/COMMITTING) blocks ordinary writers regardless of PID liveness.

POST-SUCCESS OBLIGATIONS (after COMPLETED.json written):
(f) Durable factory-artifacts commit records census stdout, activation manifest, COMPLETED.json,
    and binary version as NIST AU-9 audit record (ADR-052 §Decision 6).
(g) State-manager archives activation manifest to `.factory/migration-audit/`.

All other `.factory/` writes by agents remain subject to the Edit/Write-only constraint.
```

### Decision 9 — Resolving the S-25.06 Rule 7 contradiction (unchanged from v1.2)

S-25.06 Rule 7's original text is WITHDRAWN and replaced with:
> "The activation step is executed via the `Bash` tool with one-time interactive human approval
> at F4 activation (no standing settings.json allowlist). The migration binary is invoked at its
> absolute trusted path: `{project-root}/target/release/factory-dispatcher backfill-append-logs`
> (or `migrate-bc-index`). The invocation is NOT an Edit/Write tool call. It is a Bash execution
> of the native migration binary under ADR-052 §Decision 3's closed argument grammar, activated
> under ADR-052 §Decision 4's armed-activation manifest, producing a census report captured per
> ADR-052 §Decision 6."
Story-writer updates S-25.06 Rule 7 text accordingly.

### Decision 10 — BC-1.18.010 Invariant 2: ARCH-INDEX mapping three-way revision binding (unchanged from v1.2)

Three-way activation-time parity check: `config.arch_index_sha` ==
`manifest.approved_arch_index_sha` == live ARCH-INDEX SHA. All three must be equal; fails CLOSED
if any pair diverges. CI parity test (`arch_index_parity`) and stale-installed-config test
unchanged from v1.2. The mapping is read from deserialized config only, never from ARCH-INDEX
at runtime.

### Decision 11 — Executable verify-to-execute binding (NEW — closes F11)

**Problem:** The v1.2 design hashes `target/release/factory-dispatcher` at a path and stores a
digest, then executes that same path via a shell later. This is CWE-367 TOCTOU: between the hash
check and exec, a concurrent `cargo build` can replace the binary with legitimately different bytes.

**Fix: open once → hash through the open fd → execute the same fd (never return to pathname).**

#### Linux path (fully closes TOCTOU)

```rust
let fd = open("{project-root}/target/release/factory-dispatcher", O_RDONLY | O_CLOEXEC);
// Hash through the open fd (read() loop); compare against stored digest
let computed = sha256_read_through_fd(fd)?;
assert_eq!(computed, stored_digest, "binary integrity check failed");
// Execute through the SAME fd — no re-resolution of pathname possible
// glibc >= 2.27: execveat(fd, "", argv, envp, AT_EMPTY_PATH)
// Fallback on older glibc: fexecve(fd, argv, envp) via /proc/self/fd/<n>
execveat(fd, CStr::from_bytes_with_nul(b"\0")?, argv, envp, AT_EMPTY_PATH)?;
```

`AT_EMPTY_PATH` on Linux 3.19+ (glibc 2.34 wrapper) executes the file referred to by `fd`
without re-resolving a pathname. The binary being executed is byte-for-byte identical to the
one that was hashed.

**O_PATH caveat:** Do NOT open with `O_PATH` for this purpose — `O_PATH` fds cannot be `read()`
through, so hashing is not possible. Use `O_RDONLY`.

#### macOS path (documented residual TOCTOU — NEEDS HUMAN SIGN-OFF)

`fexecve` and `execveat AT_EMPTY_PATH` are **unavailable on macOS** (darwin-arm64). Apple's
documented exec interfaces (`execl`, `execle`, `execlp`, `execv`, `execvp`, `execvP`, `execve`)
are all pathname-based. No Apple-documented fd-binding exec primitive exists. The strongest
available evidence is Apple's exec man-page set exposing only pathname-based APIs; corroborating
non-Apple SDK/source reports confirm the missing symbol. This is not backed by an explicit
Apple negative statement, but confidence is HIGH.

**Architect decision for macOS: freeze build under maintenance lock + accept documented residual window.**

Rationale:
1. The migration binary and the verify/exec step are in the same process execution chain —
   there is no separate trusted launcher.
2. The maintenance lock (advisory flock) is held before the digest check. No agent edit/write
   paths can replace the binary (they target `.factory/`, not `target/release/`).
3. The realistic threat is a concurrent `cargo build` replacing the binary. Under normal F4
   activation procedure, no concurrent build should be running during a human-supervised,
   manually-approved migration execution. The activation procedure MUST document this constraint.
4. Protected staging (copy to `/tmp/` + `UF_IMMUTABLE`) does not provide meaningful security
   improvement: `UF_IMMUTABLE` is owner-changeable, `/tmp/` is owner-writable, and the staging
   step itself has a TOCTOU window.
5. The residual window is intra-process (sub-millisecond under quiescent system) with no
   concurrent build running.

**macOS implementation:**
```rust
#[cfg(target_os = "macos")]
fn verify_and_exec_binary(binary_path: &Path, ...) -> Result<(), ExecError> {
    let fd = File::open(binary_path)?;
    let computed = sha256_read_through_fd(&fd)?;
    if computed != stored_digest {
        return Err(ExecError::DigestMismatch);
    }
    // No fd-binding exec available on macOS; execute by pathname under held maintenance lock
    // DOCUMENTED RESIDUAL TOCTOU: a concurrent cargo build between hash and exec could
    // substitute different bytes. Mitigation: no concurrent build during F4 activation.
    // This residual risk requires human acknowledgment at ratification (see §Decision 11).
    exec_by_pathname(binary_path, ...)?
}
```

**Test:** A test MUST verify that digest mismatch (binary replaced between open and exec check)
causes the migration to abort before exec with `E-BINARY-INTEGRITY-FAILURE` rather than silently
executing the replacement. On Linux, this test also verifies the execveat AT_EMPTY_PATH path via
a test double that intercepts the execveat call and confirms the fd argument matches the opened fd.

**Human sign-off required:** The macOS residual TOCTOU window (concurrent build substitution
between hash check and `execve` pathname call) is a documented, operationally-mitigated risk.
The human must explicitly acknowledge this residual risk as part of POLICY 22 ratification of
v1.3. The migration activation procedure MUST include the constraint: "Do not run `cargo build`
while a migration is in progress." This acknowledgment must be recorded in the D-NNN entry that
ratifies ADR-052 v1.3.

---

## Rationale

### Head-to-head mechanism evaluation

#### Option A — Hook-driven explicitly-armed one-shot native action

**Viability: CONFIRMED.** Rationale unchanged from v1.2. Option A remains viable; not chosen on
operational complexity grounds. A future recurring migration use case should reconsider Option A.

#### Option B — One-time interactive Bash approval at F4 (CHOSEN)

Unchanged from v1.2. Least privilege; F4 human gate; direct D-449(a) evidence; zero standing
permission surface after migration completes.

#### Option C — Hardened standing allowlist

Unchanged from v1.2. Fails F3 (standing permission ≠ activation authorization); post-migration
cleanup burden; no governance benefit over Option B.

#### Why v1.3 atomic-pointer model was adopted over incremental v1.2 amendment

The 3rd Codex review correctly identified that F1, F2, F5, F9 are all symptoms of the
same root cause: per-file in-place rename cannot provide all-or-nothing multi-file visibility.
Incremental amendments to a per-file-rename model would continue to accrue findings at each
review cycle (confirmed by the diverging trajectory: 7→8→11). The atomic-pointer model dissolves
the shared root cause in one structural change:

- **F1** dissolves because readers now resolve CURRENT.json once and pin one generation — no
  "partially renamed" state is ever visible to a reader.
- **F2** dissolves because the intent log + matching-destination-hash recovery rule closes the
  rename/journal gap without requiring staging files to survive.
- **F5** dissolves because the txn record is separate from the flock; recovery ownership is a
  separate acquisition that does not require clearing intent.
- **F9** dissolves because COMPLETED.json is permanent and never archived; its presence is
  unambiguous in all steady states.
- **F6** collapses because there is now a SINGLE pivot point (CURRENT.json pointer swap) rather
  than N sequential renames, making the authorization gate placement unambiguous.

The git-worktree adaptation (prefer committed manifest file over directory symlink; retain fixed
canonical shard paths; use generation staging directory as intermediate read path) was chosen
over the full "generation directories as canonical storage" model to preserve shard files as
ordinary tracked git files while still providing the atomic visibility guarantee.

### Why the advisory flock replaces O_CREAT|O_EXCL + unlink-reclaim (F4, F5)

The `O_CREAT|O_EXCL` design has an inherent inode-split race: two processes can both decide a
lock file is stale, one replaces it, and the other's already-decided unlink removes the live
replacement. Stale lock reclamation via unlink is fundamentally unsafe because `unlink()` removes
a pathname, not an inode's validity. An advisory flock on a stable, never-unlinked inode has no
reclamation race: the kernel releases the lock atomically on process death or fd close, and the
next contender acquires it without any pathname manipulation. PID metadata becomes informational,
not authoritative.

### Why the authorization gate is at the CURRENT.json pointer swap, not at PREPARED→COMMITTED (F6)

v1.2 §Decision 4d placed the expiry check before the "PREPARED → COMMITTED" transition — but
that transition was itself defined as occurring AFTER all renames. An expiry check after the
first rename is on the wrong side of the pivot. The CURRENT.json pointer swap is the first
irreversible action (after it, readers see the migration as in-progress and may use generation
paths). Moving the expiry check to immediately before this single pivot is the correct
architectural position: before, expiry can safely abort with no canonical state changed; after,
expiry must not affect the ability to forward-recover.

### Why fexecve / execveat AT_EMPTY_PATH on Linux, not macOS (F11)

The research brief (§RQ4) confirms `fexecve` and `execveat AT_EMPTY_PATH` are Linux primitives
absent from Apple's documented exec API surface. The "protected staging" alternative (copy to
trusted dir + `UF_IMMUTABLE`) does not achieve fd-binding: it uses a pathname for the exec call,
the staged pathname is still subject to unlink/replace, and `UF_IMMUTABLE` is owner-changeable
without root on macOS. The freeze-build-under-maintenance-lock alternative is weaker but is more
operationally honest: it documents the residual window rather than claiming false security.

---

## Consequences

### Positive

- Resolves all 11 findings from the 3rd Codex cross-vendor closure review.
- Atomic CURRENT.json pointer swap provides genuine all-or-nothing multi-file visibility.
- COMPLETED.json as a permanent terminal record eliminates all reader/rerun steady-state
  ambiguity and survives cleanup.
- Framed checksummed intent log with matching-destination-hash recovery makes crash recovery
  both self-describing and idempotent.
- Advisory flock on stable inode eliminates lock-split race and makes stale-owner reclamation
  automatic (kernel-guaranteed on process death).
- Txn record separate from flock provides durable maintenance intent that blocks ordinary writers
  regardless of PID liveness.
- Authorization gate at single pivot point makes the authorization window unambiguous.
- Linux fd-binding exec eliminates exec TOCTOU on the primary development platform.
- Platform-branched durability barriers correctly apply F_FULLFSYNC on macOS (Apple-documented)
  vs. fsync+dir-fsync on Linux (POSIX-standard).

### Negative

- Significantly more implementation complexity than v1.2: staging generation dir, framed intent
  log, advisory flock, txn record, CURRENT.json + COMPLETED.json pointer, platform-branched
  durability, guard-branch separation, fd-binding exec.
- macOS exec TOCTOU residual window requires human acknowledgment and documented operational
  constraint at ratification.
- APFS directory-fsync durability is unverified: empirical darwin-arm64 durability test required.
- Fault injection test suite between every rename/fsync/intent-record-write step adds test scope.

### Neutral

- BC-1.18.011 Amendment 6 correction (Linux-vs-macOS dir-fsync) changes the spec but not the
  fundamental migration approach.
- The Option B selection (one-time interactive Bash approval) is unchanged.
- The three-way ARCH-INDEX parity check (§Decision 10) is unchanged.

---

## Downstream to Product-Owner

The architect specifies the following amendments; the product-owner writes all BC body changes.
Do NOT modify BC content directly; route to product-owner.

### BC-1.18.011 required amendments (v1.3 updates — ordered by precedence)

**Amendments 1–3 (scheduling coupling removal, PC6 coupling, PC7 scope):** UNCHANGED from v1.2.
Apply verbatim as specified in v1.2 §Downstream to Product-Owner Amendments 1–3.

**Amendment 4 — Precondition 5: Phase markers (v1.3 update — replaces v1.2 text):**

Replace the Amendment 4 text from v1.2 with:

> "5. A durable transaction record at `.factory/migration-state/txn-<activation_uuid>.json`
>    and a framed checksummed intent log at
>    `.factory/migration-state/intent-<generation_uuid>.log` are maintained across the full
>    migration lifecycle:
>    - State STAGING: flock held; quiescence reached; staging generation built; intent log
>      written with per-target expected hashes + pre-states; all fsync barriers applied;
>      authorization gate check pending.
>    - State COMMITTING (the pivot): CURRENT.json pointer swap executed atomically; from this
>      point forward recovery is mandatory; authorization expiry does NOT abort.
>    - State COMPLETED: all canonical path moves complete and hash-verified; COMPLETED.json
>      written at stable path; PERMANENT.
>    EC-003 resume logic reads the intent log + txn record to determine which canonical path
>    moves succeeded (matching-destination-hash rule) and resumes from first uncompleted move."

**Amendment 5 — Precondition 6: Writer exclusion / maintenance boundary (v1.3 update):**

Replace the Amendment 5 text from v1.2 with:

> "6. A WRITER-EXCLUSION maintenance boundary is in force during migration execution via two
>    independent mechanisms:
>    (a) Advisory flock on `.factory/migration-state/exclusive.lock` (pre-created, never unlinked):
>        the migration binary holds an exclusive flock for the full execution window; kernel
>        releases automatically on process death; stale-owner detection is automatic.
>    (b) Txn record at `.factory/migration-state/txn-<uuid>.json` with state STAGING or COMMITTING:
>        ALL mutation tool calls (Edit/Write/MultiEdit/Bash) targeting BC-INDEX paths are blocked
>        by the native admission gate in `executor.rs` (ADR-052 §Decision 5a) when a txn record
>        exists in STAGING or COMMITTING state — regardless of whether the flock is currently held.
>        This ensures ordinary writers remain blocked even during crash recovery when no process
>        holds the flock.
>    (c) OPEN/DRAINING gate with writer reservations spanning PreToolUse→tool-completion ensures
>        the migration coordinator waits for all in-flight admitted writers to complete before
>        snapshotting source files (ADR-052 §Decision 5a)."

**Amendment 6 — Postcondition 3: Dir-fsync mandate (v1.3 CORRECTION — replaces v1.2 text):**

Replace the Amendment 6 text from v1.2 (which incorrectly treated dir-fsync as mandatory on all
platforms) with:

> "Each atomic file replacement (`rename(2)` call) MUST be followed by a platform-appropriate
> durability barrier before proceeding to the next replacement:
> - Linux (ext4/xfs): `fsync(file_fd)` + `fsync(parent_dir_fd)` — mandatory; ensures directory
>   entry survives a system crash per Pillai et al. OSDI'14.
> - macOS/APFS: `fcntl(file_fd, F_FULLFSYNC)` — mandatory for power-loss durability (Apple
>   `fsync(2)` does NOT flush the drive cache; `F_FULLFSYNC` is the documented durability lever);
>   `fsync(parent_dir_fd)` — best-effort only; Apple docs do not guarantee APFS directory-fsync
>   provides power-loss durability.
> This platform-branched durability guarantee is implemented in `sync_file_durable()` and
> `sync_dir_best_effort()` per ADR-052 §Decision 7d."

**Amendments 7, 8, 9 (TOCTOU guard, Invariant 3 commit-pointer, Architecture Anchors):**
Replace v1.2 references to COMMITTED marker, `completed_renames`, and per-file rename sequence
with references to: CURRENT.json pointer swap (commit point), intent log + matching-hash
recovery, COMPLETED.json (permanent terminal record), and txn record state machine. Architect
will provide exact replacement text in the next burst dispatch to product-owner.

### BC-1.18.010 required amendments (v1.3 updates)

**Invariant 2 amendment:** UNCHANGED from v1.2. Apply verbatim.

**Reader integration amendment (v1.3 update — replaces v1.2 text):**

Replace the v1.2 §Reader Integration section with:

> "During the B2 migration window (after CURRENT.json pointer swap, before COMPLETED.json
> written), readers accessing BC-INDEX paths MUST use the following protocol:
> 1. Check `.factory/migration-state/completed.json` — if exists: canonical paths are current.
> 2. Check `.factory/migration-state/CURRENT.json` — if `status: committing`: use
>    `.factory/migration-state/gen-<generation_id>/` paths for reads.
> 3. If neither exists: legacy BC-INDEX.md path is current (migration not started).
> In steady state (COMPLETED.json present), canonical paths are always authoritative. The
> 'COMMITTED absent → read legacy BC-INDEX.md' heuristic from v1.2 is eliminated: COMPLETED.json
> is permanent and its presence is unambiguous in all states including after cleanup."

### error-taxonomy.md correction (v1.3 update — replaces v1.2 text)

Replace the v1.2 correction with:

> "The maintenance lock (E-MAINTENANCE) is enforced by two independent mechanisms:
> (1) The native admission gate in `executor.rs` (ADR-052 §Decision 5a) blocks ALL mutation tool
>     calls (Edit, Write, MultiEdit, Bash) targeting BC-INDEX paths when a txn record at
>     `.factory/migration-state/txn-*.json` exists with state STAGING or COMMITTING — this check
>     is independent of PID liveness and fires before any registry plugin.
> (2) The OPEN/DRAINING gate state prevents new writer reservations when the maintenance
>     coordinator has declared maintenance intent (gate state = DRAINING or LOCKED).
> The `validate-factory-path-staging` guard (Bash path) is a secondary classifier layer that
> enforces the full-command classifier and four-branch guard logic (ADR-052 §Decision 5c)."

---

## BC Impact for Product-Owner Re-hardening

The v1.3 architectural changes require the following specific changes to BC-1.18.010,
BC-1.18.011, and error-taxonomy.md. This section is the authoritative handoff to the
product-owner. DO NOT edit BC bodies directly — route all changes via product-owner.

### BC-1.18.011 — what must change to match v1.3

| Section | v1.2 text (to replace) | v1.3 change required |
|---|---|---|
| Precondition 5 (§Amendment 4) | PREPARED/COMMITTED/CLEANED phase markers; `completed_renames` tracking; EC-003 reads `completed_renames` | Replace with: STAGING/COMMITTING/COMPLETED txn record; intent log for recovery; EC-003 reads intent log + txn record |
| Precondition 6 (§Amendment 5) | O_CREAT\|O_EXCL lock file with alive-PID check; Edit/Write gate via `validate-factory-path-staging` only | Replace with: advisory flock on stable inode; txn record state blocks writers regardless of PID liveness; OPEN/DRAINING gate with writer reservations |
| Postcondition 3 (§Amendment 6) | "dir-fsync is mandatory, not best-effort" (applies to all platforms) | Replace with: Linux mandatory dir-fsync; macOS F_FULLFSYNC on file mandatory; dir-fsync best-effort only on APFS |
| Postcondition 3a (§Amendment 7) | Single TOCTOU check before first rename; abort leaves BC-INDEX.md untouched | Update: fingerprint check still single; but now occurs before CURRENT.json pointer swap (step 5 in §Decision 7c), not before first rename |
| Invariant 3 (§Amendment 8) | "COMMITTED marker is the sole commit-point"; references `completed_renames` | Replace with: CURRENT.json pointer swap (atomic rename) is the sole commit-point; forward recovery uses intent log + matching-hash rule |
| Architecture Anchors (§Amendment 9) | References §Decision 7a PID+activation_id lock file; §Decision 7 per-file rename sequence | Replace with: §Decision 7a advisory flock + txn record; §Decision 7b intent log; §Decision 7c CURRENT.json pointer swap + COMPLETED.json |

### BC-1.18.010 — what must change to match v1.3

| Section | v1.2 text (to replace) | v1.3 change required |
|---|---|---|
| §Reader Integration | "COMMITTED absent → read legacy BC-INDEX.md"; COMMITTED archived at CLEANED | Replace entire section with v1.3 reader protocol (COMPLETED.json check first; CURRENT.json generation pinning second; legacy only if neither exists) |
| §Reader Integration steady state | "after CLEANED, shard paths are canonical; COMMITTED archived" | Replace: COMPLETED.json is permanent + authoritative; no archiving of any terminal record |
| Invariant 3 (if present) | References COMMITTED marker as commit-pointer | Update to: CURRENT.json pointer swap is the commit-point; COMPLETED.json is the terminal record |

### error-taxonomy.md — what must change to match v1.3

| Location | v1.2 text (to replace) | v1.3 change required |
|---|---|---|
| E-MAINTENANCE definition (near original line 84) | "enforced by `validate-factory-path-staging` on Edit/Write"; "when exclusive.lock exists with alive PID" | Replace with: dual-mechanism: (1) txn record state STAGING/COMMITTING blocks all mutation tools via native gate regardless of PID; (2) DRAINING gate prevents new reservations; guard is secondary classifier only |

---

## References

- `adv-cv-adr052-v12-closure-2026-09-13.md` — 3rd Codex cross-vendor closure review (11 findings, D-1218) that prompted this v1.3 redesign
- `research-adr-052-v13-atomic-publication-2026-09-13.md` — research brief grounding this v1.3 architecture (atomic publication, intent log, advisory flock, F_FULLFSYNC, fexecve/execveat)
- `adv-cv-adr052-v11-closure-2026-09-12.md` — 2nd Codex closure review, 8 findings (D-1216)
- `adv-cv-adr052-cluster5-F1-2026-09-12.md` — 1st Codex RATIFY-WITH-CHANGES verdict (7 findings, D-1214)
- `research-adr-052-assumption-validation-2026-09-12.md` — prior research Q1-Q5 validation
- `BC-1.18.011` — migration BC; §Downstream to Product-Owner specifies amendments
- `BC-1.18.010` — §Reader Integration amendment + Invariant 2 amendment
- `S-25.06-append-log-backfill-split-executor.md` Rule 7 — corrected per §Decision 9
- `ADR-051` §Decision 1 (WASM fuel-budget constraint), §Decision 7 (B2 end-state)
- `CLAUDE.md` — TD-FACTORY-HOOK-BYPASS-001 P0 governing rule; human-only edit target
- POSIX rename(2) [man7.org]: namespace atomicity guarantee; same-filesystem constraint; NOT multi-file transaction
- Pillai et al. OSDI'14 "All File Systems Are Not Created Equal": crash vulnerabilities across 6 Linux filesystems; dir-fsync requirement for rename durability
- Apple fsync(2) [Apple Developer]: "the drive itself may not physically write the data to the platters for quite some time" — F_FULLFSYNC required for APFS power-loss durability
- Apple fcntl(2) [Apple Developer]: `F_FULLFSYNC` asks drive to flush all buffered data; `F_BARRIERFSYNC` is ordering-only
- man7 flock(2), fcntl(2): advisory lock semantics; OFD lock on Linux; `F_SETLK` any-close hazard
- man7 execveat(2), fexecve(3): fd-binding exec on Linux; `AT_EMPTY_PATH` for executing by fd
- LMDB / SQLite super-journal: prior art for single-pointer atomic commit over multi-file publication
- Kleppmann 2016 / Chubby OSDI'06: lease + fencing tokens for distributed locking
- CWE-367 (TOCTOU), CWE-88 (argument injection), CWE-78 (OS command injection), CWE-22 (path traversal)
- NIST SP 800-53r5 AU-9 (audit trail tamper-evidence)

## Files to Change

| File | Change | Owner |
|---|---|---|
| `CLAUDE.md` | Apply §Decision 8 CLAUDE.md amendment text | **Human only** |
| `.claude/settings.json` | No change — Option B requires no settings.json mutation | — |
| `.factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md` | Apply §Downstream Amendments 1–9 (v1.3 versions) | product-owner |
| `.factory/specs/behavioral-contracts/ss-01/BC-1.18.010.md` | Apply §Downstream Invariant 2 amendment + §Reader Integration v1.3 replacement | product-owner |
| `.factory/specs/prd-supplements/error-taxonomy.md` | Apply §Downstream error-taxonomy v1.3 correction | product-owner or technical-writer |
| `.factory/migration-state/exclusive.lock` | Create empty pre-seeded flock inode (never deleted) | devops-engineer (cluster-5 activation preparation) |
| `plugins/vsdd-factory/hooks-registry.toml` | Amend 4 guards per §Decision 5b (txn record state awareness replacing PID-liveness check) | devops-engineer |
| `plugins/vsdd-factory/hooks/destructive-command-guard.sh` (or WASM) | 4-branch classifier per §5c; txn record state check | devops-engineer |
| `plugins/vsdd-factory/hooks/validate-factory-path-staging.sh` (or WASM) | 4-branch classifier; conservative Bash admission; executable digest verification + fd-binding per §Decision 11 | devops-engineer |
| `plugins/vsdd-factory/hooks/validate-factory-path-staged.sh` (or WASM) | Recognize governed migration subcommands; pass through | devops-engineer |
| `crates/factory-dispatcher/src/executor.rs` | OPEN/DRAINING gate with writer reservations (PreToolUse-acquire/PostToolUse-release); txn record state check blocking mutations | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/src/shard_manager.rs` | Full v1.3 migration implementation: advisory flock; txn record; intent log (framed+checksummed); CURRENT.json pointer swap; COMPLETED.json; 4-branch recovery; platform-branched durability (F_FULLFSYNC on macOS); fd-binding exec on Linux; freeze-build on macOS with documented residual | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/tests/` | v1.3 test suite: advisory flock acquisition/stale-reclamation; txn record state machine; intent log write+recovery+fault-injection; CURRENT.json pointer swap atomicity; COMPLETED.json permanence; 4-branch guard logic; conservative Bash admission; digest mismatch abort before exec; Linux execveat AT_EMPTY_PATH path; macOS freeze-build documented residual; platform durability branches; F_FULLFSYNC on macOS | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/tests/darwin_arm64_durability_test.rs` | Empirical darwin-arm64 directory-fsync durability characterization test (required to validate APFS dir-fsync behavior; flags whether dir-fsync provides any power-loss protection beyond F_FULLFSYNC alone) | implementer (cluster-5 TDD) — darwin-arm64 CI required |
| `.factory/activation/factory-dispatcher.sha256` | SHA-256 of built binary | devops-engineer (cluster-5 activation) |
| `.factory/activation/` | Created at F4 by state-manager | state-manager |
| `.factory/migration-state/` | Created by migration binary at runtime; `exclusive.lock` pre-seeded by devops-engineer | migration binary (runtime) |
| `.factory/migration-audit/` | Created by state-manager post-migration | state-manager |

## Changelog

| Version | Date | Author | Change |
|---|---|---|---|
| 1.3 | 2026-09-13 | architect | Full redesign per D-1218 (3rd Codex cross-vendor closure review, 11 findings, NOT RATIFIABLE). Adopts atomic-pointer architecture from research-adr-052-v13-atomic-publication-2026-09-13.md. Closes all 11 findings: F1/F9 — single atomic CURRENT.json pointer swap over immutable staging generation; COMPLETED.json as permanent terminal record; eliminates ambiguous-absence heuristic. F2 — framed checksummed intent log with per-target expected post-hash + pre-state; matching-destination-hash recovery; fail-closed recovery decision table; fault-injection test mandate. F3 — real OPEN/DRAINING admission gate with PreToolUse-acquire/PostToolUse-release writer reservations; wait for active_writer_count=0 before snapshot; no non-atomic shared→exclusive flock upgrade. F4 — advisory flock on stable pre-created never-unlinked inode; fail-closed on empty/corrupt metadata by acquire-first; automatic stale reclamation via kernel on process death. F5 — durable txn record separate from flock; ordinary writers blocked by txn state (STAGING/COMMITTING) regardless of PID liveness; recovery-owner bumps fencing_generation to claim ownership; releasing flock never deletes txn record. F6 — authorization gate moved to immediately before CURRENT.json pointer swap (single pivot); post-pivot COMMITTING state retained regardless of expiry; completion-only recovery manifest bound to old activation_id + staged-generation hash + fencing generation (replaces fresh activation). F7 — conservative Bash admission: block unknown-write-effect commands while txn record in STAGING/COMMITTING; classify sanctioned commands from config-driven target sets; canonicalization + alias rejection. F8 — four guard branches: census (no manifest), terminal-state no-op (COMPLETED.json check), new activation (full validation), recovery (completion-only manifest); manifest/flock required only for new activation and recovery branches. F10 — single authoritative allowed-write-targets list; CLAUDE.md amendment generated from that exact list with containment check at every write; sub-shards (.a.md/.b.md), BC-INDEX.shard-manifest.toml, BC-INDEX-SS-05.manifest.toml added. F11 — Linux: fexecve/execveat(AT_EMPTY_PATH) for fd-binding exec; macOS: fexecve UNAVAILABLE per Apple docs; decision: freeze build under maintenance lock + documented residual TOCTOU; human sign-off required at ratification. Platform durability: Linux fsync+dir-fsync (mandatory); macOS F_FULLFSYNC on file (mandatory per Apple docs); APFS directory-fsync best-effort only (Apple docs inconclusive); darwin-arm64 empirical durability test required. Corrects v1.2 Amendment 6 which incorrectly treated dir-fsync as mandatory on all platforms. |
| 1.2 | 2026-09-13 | architect | Full redesign per D-1216 (2nd Codex RATIFY-WITH-CHANGES 8 findings). Resolves: F1 — native admission gate added in `executor.rs` covering ALL mutation tools (Edit/Write/MultiEdit/Bash) before shard_cap_precheck, replacing the `^Bash$`-only guard-level check; drain protocol via TOCTOU abort. F2 — per-target completion tracking in PREPARED marker + single TOCTOU check before first rename (not repeated between renames) resolves COMMITTED-after-last-rename vs "original untouched" contradiction; reader integration protocol specified (COMMITTED marker as read-path selector for BC-1.18.010). F3 — content-bearing lock file with PID+activation_id separates persistent maintenance intent from OS advisory lock; pre-PREPARED crash recovery path defined; stale-lock recovery specified. F4 — two-phase manifest validation: pre-lock (lightweight) then under-exclusion (repo_root_sha, expected_total_bcs, three-way ARCH-INDEX parity, readiness); pre-publication expiry recheck before COMMITTED; manifest CONSUMED marking replaces deletion (resolves "absent=rejected" vs "already-migrated=exit0" contradiction); three expiry-safe recovery modes defined. F5 — explicit three-way activation-time parity check: config.arch_index_sha == manifest.approved_arch_index_sha == live ARCH-INDEX SHA; catches stale-binary case (revision A config + revision B manifest + revision B live). F6 — accurate skipped-control inventory: brownfield-discipline description corrected to ".reference/ write protection"; factory-branch-guard row added with explicit waiver; validate-factory-path-staged PostToolUse Bash corrected from "bypassed" to "MUST be amended." F7 — full-command pre-shell classifier moved to guard layer (validate-factory-path-staging + destructive-command-guard) with executable digest verification; removed impossible "binary rejects metacharacters post-shell-parse" claim; negative tests specified. F8 — CLAUDE.md amendment text expanded to include BC-INDEX.md + migration-state/ + activation/ + migration-audit/ + config-specified mech-A targets; preconditions (a-e) separated from post-success obligations (f-g); CLAUDE.md rule referenced by text anchor not line number; 4 guard amendments listed (vs 3 in v1.1). Downstream to Product-Owner: Amendment 5 corrected (Edit/Write → ALL mutation tools via native admission gate); error-taxonomy.md correction added; BC-1.18.010 §Reader Integration section added. NOT RATIFIED per D-1218. |
| 1.1 | 2026-09-12 | architect | Full revision per D-1214 (1st Codex RATIFY-WITH-CHANGES 7 findings + research Q1-Q5). Mechanism changed from Option C (hardened standing allowlist) to Option B (one-time interactive Bash approval at F4, no settings.json change). Option A re-evaluated honestly. Declares explicit narrowly-authorized policy exception (§Decision 8). Activation-manifest authorization mechanism added (§Decision 4). 9 PreToolUse `^Bash$` dispatcher guards enumerated (§Decision 5). Audit trail upgraded to factory-artifacts commit satisfying NIST AU-9 (§Decision 6). Crash-atomicity phase markers, TOCTOU pre-commit guard, dir-fsync mandate added (§Decision 7). Config snapshot bound to ARCH-INDEX revision with activation-time parity check (§Decision 10). BC-1.18.011 Amendments 1-9 and BC-1.18.010 Invariant 2 amendment specified. NOT RATIFIED per D-1216. |
| 1.0 | 2026-09-12 | architect | Initial authoring. Resolves CV-DIR-F2 (impossible execution path). Specifies Bash-tool-with-allowlist as sanctioned invocation path for both mechanism-A (S-25.06) and mechanism-B2 (BC-1.18.011) migrations. NOT RATIFIED per D-1214. |
