---
document_type: architecture-decision-record
adr_id: ADR-052
level: L3
version: "1.6"
status: proposed
date: 2026-09-13
producer: architect
timestamp: 2026-09-13T00:00:00Z
phase: F1
supersedes: null
superseded_by: null
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
  - .factory/cycles/v1.0-brownfield-backfill/adv-local-adr052-pass3.md
input-hash: "0ae730c"
# input-hash: run compute-input-hash --update at state-manager registration burst
---

# ADR-052: Native Migration CLI — Sanctioned Execution Path for Governed One-Time Shard Migrations

## Status

PROPOSED — v1.5 NOT ratified per 6th adversarial review (local cascade pass-3, D-1223)
(1 CRITICAL + 4 HIGH + 5 MEDIUM + 2 LOW findings; step 3b census gate was a tautology
against its own intent-log hash; §4e table keyed on impossible "CURRENT.json status:staging"
discriminator (CURRENT.json first written at step 6); completion-crash leaves gate
permanently LOCKED because Branch 2 ALREADY_MIGRATED acquires no lock; Source/Origin and
References cited non-existent adv-cv-adr052-v13-closure-2026-09-13.md; §Downstream
Amendments 7/8/9 deferred with placeholder text). This v1.6 fix-burst resolves all
findings at prose, code-sample, and predicate levels. Human POLICY 22 ratification
required before this ADR is treated as accepted. Cluster-5 TDD dispatch remains BLOCKED
until ratification.

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

## Finding → Resolution (v1.3 + v1.4)

All 11 findings from the 3rd Codex cross-vendor closure review (`adv-cv-adr052-v12-closure-2026-09-13.md`) are resolved by the v1.3 redesign. Nine additional findings from the 4th adversarial review (local cascade pass-1, D-1221) are resolved by this v1.4 fix-burst:

| Finding | Sev | Root cause in v1.2 | Resolution in v1.3 |
|---------|-----|-------------------|-------------------|
| F1 | HIGH | Per-file rename: reader between renames sees mixed generation; "COMMITTED absent → read legacy" heuristic is ambiguous after crash | §Decision 7c: single atomic CURRENT.json pointer swap; reader resolves pointer once, pins one generation; completed.json as permanent terminal record eliminates ambiguous-absence heuristic |
| F2 | HIGH | Crash after rename but before `completed_renames` append: staging gone, no record, retry cannot recover | §Decision 7b: framed checksummed intent log records per-target expected post-content hash + expected pre-state BEFORE first rename; recovery accepts matching destination hash as completion; fail closed on ambiguity |
| F3 | HIGH | "Drain protocol" only detects writes already completed; admits writers before lock; nothing waits for in-flight writer to finish | §Decision 5a: real OPEN/DRAINING gate; writer reservations span PreToolUse→tool-completion; coordinator waits for active_writer_count=0 before snapshotting; fingerprint recheck kept as defense-in-depth |
| F4 | HIGH | O_CREAT\|O_EXCL + unlink-reclaim: two reclaimers both classify inode as stale, one unlinks the other's live inode; empty/corrupt lock classified as "dead → unlink" | §Decision 7a: advisory flock(LOCK_EX) on stable pre-created never-unlinked inode; stale reclamation automatic on process death; fail closed on empty/corrupt metadata by acquire-first (never unlink) |
| F5 | HIGH | No exclusive takeover path for dead-PID STAGING transaction; BC-1.18.011 and error-taxonomy.md block writers only for alive PID, contradicting "block for any STAGING" | §Decision 7a: durable txn record separate from flock; ordinary writers blocked by txn record state (STAGING or COMMITTING) regardless of PID liveness; recovery-owner bumps fencing generation to claim ownership; releasing flock never deletes txn record |
| F6 | HIGH | Expiry recheck placed after all renames (§4d in v1.2); expiry can delete PREPARED after canonical content already changed; no authorized path for new activation to recover old transaction | §Decision 4d: authorization gate moved to immediately before CURRENT.json pointer swap (single pivot); post-pivot COMMITTING state retained regardless of expiry; completion-only recovery manifest bound to old activation_id + staged-generation hash + monotonic fencing generation |
| F7 | HIGH | Admission gate conditions on "target path under protected directory" for Bash, but Bash commands yield no intrinsic target paths; unknown write effects undefined | §Decision 5c: conservative Bash admission block: while txn record exists in STAGING/COMMITTING state, block all Bash commands with unknown write effects; classify only sanctioned migration commands from config-driven target sets; canonicalization and alias rejection specified |
| F8 | MED | Guard requires unexpired manifest for --census (no manifest needed) and COMMITTED reruns (terminal no-op needs no manifest); both paths blocked after manifest archival | §Decision 5c: four separate guard branches: read-only census, validated terminal-state no-op, new activation, and recovery; manifest requirement and flock acquisition apply only to new-activation and recovery branches |
| F9 | HIGH | After CLEANED archives COMMITTED, absent COMMITTED is indistinguishable between "never ran" and "completed long ago"; reader and rerun logic cannot determine steady state | §Decision 7c: completed.json written at stable path after all target moves; permanent, never deleted, never archived; supersedes CURRENT.json for reader protocol; closes steady-state ambiguity |
| F10 | HIGH | Allowed-paths list in §Decision 8 omits sub-shards (.a.md/.b.md), BC-INDEX.shard-manifest.toml, BC-INDEX-SS-05.manifest.toml; CLAUDE.md amendment uses "shards/" directory which is broader scope | §Decision 8: single authoritative allowlist enumerating all required migration output paths including sub-shards and manifest files; CLAUDE.md amendment generated from that exact same list with containment checks |
| F11 | HIGH | Guard hashes binary at path, then shell executes same path by name; concurrent cargo build can replace binary between hash check and exec; TOCTOU window implicit | §Decision 11: Linux: open fd, hash through fd, execveat(fd, "", argv, envp, AT_EMPTY_PATH) — eliminates TOCTOU; macOS: fexecve unavailable per Apple documentation; decision: freeze build under maintenance lock + document residual window; requires human sign-off (see §Decision 11) |

**v1.4 findings (4th adversarial review — this fix-burst):**

| Finding | Sev | Root cause in v1.3 | Resolution in v1.4 |
|---------|-----|-------------------|-------------------|
| C1 | CRITICAL | Reader no-content window: step 6 flips CURRENT.json to `committing` before step 7 renames files out of `gen-<uuid>/`; reader protocol says use `gen-<uuid>/` during committing, but those files are being renamed away → ENOENT; also violates "IMMUTABLE" invariant claim | §Decision 7c: reader protocol changed to canonical-first / generation-dir-fallback during `committing`; each file is always accessible at canonical path (if already moved) or `gen-<uuid>/` path (not yet moved) due to rename atomicity; "IMMUTABLE" clarified to CONTENT-IMMUTABLE — content never modified, but files may be moved to canonical paths during step 7 |
| C2 | CRITICAL | F10 allowlist still not closed: sub-shard names use hyphen-uppercase (`-A.md`, `-B.md`) but canonical ADR-051 + BC-1.18.010/011 use dot-lowercase (`.a.md`, `.b.md`); SS-05 needs `.c` shard (661 BCs over cap) — allowlist stops at `-B`; only `BC-INDEX-SS-05.manifest.toml` hardcoded — `BC-INDEX-SS-06.manifest.toml` rejected; `validate_write_target()` would reject all canonical sub-shard names | §Decision 8: allowlist normalized to dot-lowercase `BC-INDEX-SS-<NN>.a.md` / `.b.md` / `.c.md` (any single lowercase letter a-z); sub-manifest generic `BC-INDEX-SS-<NN>.manifest.toml` (covers all subsystems); CLAUDE.md amendment regenerated from corrected list; ratification-time test mandate: every BC-1.18.010/011 path passes `validate_write_target()` |
| H1 | HIGH | Abort leaves gate stuck LOCKED: drain-timeout (§5a step 3), expiry abort (§7c step 4), fingerprint abort (§7c step 5), census abort (§7c step 3b) all release flock but never reset gate → OPEN; crash-recovery with no live txn also leaves stale LOCKED/DRAINING forever | §Decision 5a: every abort/rollback path atomically flips gate → OPEN before returning; crash-recovery: if recovery process acquires flock and finds gate=LOCKED/DRAINING with no active txn (absent, COMPLETED, or ABORTED), MUST reconcile gate → OPEN; fault-injection test mandate: "abort during DRAINING/LOCKED restores OPEN" |
| H2 | HIGH | Drain quiescence unsound across processes: §5a persists only gate_state, not active_writer_count; dispatcher is per-event binary (not persistent daemon), so in-process mutex/condvar cannot observe cross-process reservations; no atomic check-gate-and-increment | §Decision 5a: dispatcher per-event process model explicitly stated; active_writer_count replaced by durable reservation directory `.factory/migration-state/reservations/`; PreToolUse admission is atomic (read gate_state file under lock, create reservation file, release); coordinator polls reservation dir for quiescence; test: writer admitted just before DRAINING → coordinator waits |
| H3 | HIGH | §7c sequence dropped BC-1.18.011 PC1 content-preservation + PC2 independent census pre-pivot gate; no step verifies partition correctness before the irreversible pointer swap; E-SHD-005 not checked before pivot; resume-from-STAGING (§4e) does not re-run census (EC-003) | §Decision 7c: new step 3b (pre-pivot content-preservation + census gate) inserted between intent-log WAL (step 3) and authorization gate (step 4); verifies content-preservation (PC1), independent census (PC2), and E-SHD-005 shard boundary invariants against staged generation; on failure: ABORT cleanly (delete gen, txn → ABORTED, gate → OPEN); §4e resume-from-STAGING: RE-RUN full census (EC-003) before proceeding |
| H4 | HIGH | §5c conservative Bash admission and Finding→Resolution rows F5, F7 key on txn state `PREPARED` which does not exist in §7a enum; §7a defines `STAGING\|COMMITTING\|COMPLETED\|ABORTED`; §5c Bash classifier never fires during STAGING | §Decision 5c: all `PREPARED` occurrences replaced with `STAGING`; F5/F7 rows in Finding→Resolution table updated; §7a enum is authoritative: no PREPARED state exists in this ADR |
| M1 | MED | Fencing token never enforced: §7a recovery-owner writes carry `fencing_generation` "to prove authority" but no resource rejects a stale token; only the completion manifest checks it; enforcement language implies a distributed-system fence that is not implemented | §Decision 7a: `fencing_generation` downgraded to AUDIT-ONLY metadata; advisory flock provides actual mutual exclusion (only one process can hold LOCK_EX at a time); `fencing_generation` is a monotonic traceability counter for crash-restart audit trail; "prove authority" language removed |
| M2 | MED | Pivot step-number contradiction: §4d cites "§7c step 4" as the pointer swap / single pivot; §7c labels step 4 = authorization gate and step 6 = commit/pointer swap; BC-1.18.011 PC3a additionally cites "step 5" (all three citations disagree) | §Decision 4d: corrected to reference "§7c step 6" as the CURRENT.json pointer swap (commit point); authorization gate = step 4, fingerprint recheck = step 5, pointer swap = step 6; BC-1.18.011 PC3a cites step 5 → must be corrected to step 6 (see BC impact handoff) |
| M5 | MED | macOS risk framing: §Decision 11 / Consequences say TOCTOU is "eliminated on the primary development platform (Linux)" but the primary operator platform IS macOS/darwin-arm64; "no concurrent cargo build" is a soft recommendation, not a programmatic guard | §Decision 11: macOS/darwin-arm64 identified as primary operator platform carrying the documented residual TOCTOU; Linux is where TOCTOU is eliminated via fd-binding; "no concurrent cargo build" elevated to hard pre-flight checklist item with programmatic mtime guard; /proc dependency for Linux fexecve fallback noted; §Consequences updated accordingly |

**v1.5 findings (5th adversarial review — this fix-burst):**

| Finding | Sev | Root cause in v1.4 | Resolution in v1.5 |
|---------|-----|-------------------|-------------------|
| C-1 | CRITICAL | Reader protocol regression: canonical-first/generation-fallback is correct for net-new shard files but WRONG for BC-INDEX.md (in-place overwrite target); canonical path holds OLD monolithic body throughout COMMITTING window until step 7's final rename, so canonical-first returns stale content for BC-INDEX.md before its rename; violates BC-1.18.010/011 Invariant 3 | §Decision 7c: reader protocol inverted to generation-first/canonical-fallback — try gen-\<uuid\>/\<file\> FIRST (present ⟹ not-yet-moved ⟹ new content); on absence fall back to canonical (absent from gen ⟹ already moved ⟹ new content); correct for BOTH net-new shard files AND in-place-overwrite targets; ENOENT impossible for any file during COMMITTING window; fault-injection reader test mandate added |
| H-1 | HIGH | ADR not self-contained: §Decision 6 says "(See v1.2 §Decision 6 — no change)"; §Decision 8 says "See v1.2 §Decision 8 skipped-control inventory table"; §Downstream says Amendments 1–3 "UNCHANGED from v1.2. Apply verbatim as specified in v1.2 §Downstream"; v1.2 no longer exists in the file — ratifier/PO cannot read load-bearing content | §Decision 6: full audit-trail decision inlined; §Decision 8: full skipped-control inventory table inlined; §Downstream: Amendments 1–3 fully inlined; no "see v1.2 §" dangling references remain |
| H-2 | HIGH | macOS mtime guard inert on successful exec: §Decision 11 records mtime "immediately after the digest check" and re-checks "after exec returns" — a successful execve never returns (it replaces the calling process); guard fires only on exec failure; successful substitution+exec (the dangerous case) is never caught | §Decision 11: mtime re-stat moved to IMMEDIATELY BEFORE the execve call; window narrowed from (digest-check → exec) to (pre-exec-re-stat → execve-syscall); corrected residual-risk text states guard detects substitution before the pre-exec re-stat, not after; "Human sign-off required" updated accordingly |
| H-3 | HIGH | Cross-process reservation UUID uncorrelated: PreToolUse generates a fresh random UUID for the reservation filename; PostToolUse (separate process) cannot recompute a random value from a prior process → reservation leaks → quiescence never achieved → spurious DRAIN_TIMEOUT_ABORT | §Decision 5a: reservation filename changed to `<tool_use_id>.reservation` using the stable harness tool-invocation identifier shared across the Pre/Post hook pair for the same tool call; test mandates added: Pre creates, Post (separate process, same tool_use_id) removes; stale-PID cleanup reclaims dead creator's reservation |
| H-4 | HIGH | Load-bearing ADR version pins in body narrative: CLAUDE.md amendment text says "ADR-052 v1.4 EXCEPTION"; dangling "see v1.2 §" references (addressed by H-1); version-history form belongs in changelog only | CLAUDE.md amendment text updated to "ADR-052 EXCEPTION" (stable, version-agnostic); all "see v1.2 §" dangling refs inlined (H-1); BC-impact handoff directs PO to use stable §Decision N form in BC-1.18.010/011 + error-taxonomy |
| M-2 | MED | ADR-051 §Decision 10 still says SS sub-sharding happens "at the SAME F4 activation moment mechanism A's own backfill (BC-1.18.008) runs," contradicting BC-1.18.011 Precondition 4 (decoupled, any order) and ADR-052 §Decision 1 | ADR-051 §Decision 10 item 6 amended to remove activation-moment coupling; B2 sub-split runs within same one-time B2 operation independently of mechanism A's activation schedule; ADR-051 bumped to v1.14 |
| M-3 | MED | ADR uses bare `E-MAINTENANCE` throughout; taxonomy canonical code is `E-MAINTENANCE-001` (error-taxonomy.md is SoT for codes) | `E-MAINTENANCE` replaced with `E-MAINTENANCE-001` throughout the ADR body, §5a drain language, §7a ordinary-writer blocking, and §Error Code Semantics table |
| M-4 | MED | §Downstream BC-1.18.010 §Reader Integration carries stale "use gen-\<uuid\>/ paths for reads" v1.3 instruction alongside the v1.4 canonical-first correction; two conflicting instructions present; neither is the correct generation-first protocol | §Downstream BC-1.18.010 §Reader Integration: stale v1.3 text deleted; v1.4 canonical-first text replaced; single correct generation-first/canonical-fallback instruction retained, consistent with §Decision 7c C-1 fix |
| M-6 | MED | §5a admits Bash mutations but never states when a Bash invocation creates/holds a durable reservation or whether quiescence waits on it; underspecified for the coordinator | §Decision 5a: explicit text added: every admitted Bash with write effect creates a `<tool_use_id>.reservation` file; §5c classifier determines write effect; quiescence waits for all Bash reservations; test mandate added |

**v1.6 findings (6th adversarial review — local cascade pass-3, D-1223 — this fix-burst):**

| Finding | Sev | Root cause in v1.5 | Resolution in v1.6 |
|---------|-----|-------------------|-------------------|
| C-1 | CRITICAL | Step 3b census gate is a tautology: step 3 sets `expected_post_hash = sha256(staging_file)`; step 3b PC1 checks `sha256(staged_file) == expected_post_hash` = sha256(x)==sha256(x) — always true, inert as a correctness gate. Does NOT perform BC-1.18.011 PC1 (byte-for-byte concat against `source_sha256`) nor PC2 (per-ID exactly-one-shard set membership). EC-001 (byte-identical concat with compensating dup+drop, same total count) passes undetected. | §Decision 7c step 3b: existing sha256(staged)==expected_post_hash check renamed "staging-file integrity" (what it actually is — guards against silent modification after staging sync); new PC1 added (reconstruct concatenation of staged shard files + retained lean body → compare SHA-256 against `source_sha256` from txn record); PC2 rewritten as per-ID set check (fresh enumeration of ID set from original-census; each ID must appear in EXACTLY ONE staged shard, ZERO in retained body; count comparison is necessary but not sufficient — EC-001 MUST abort). |
| H-1 | HIGH | §4e recovery table keyed on "CURRENT.json status:staging" — impossible discriminator. CURRENT.json does not exist during STAGING (first written at step 6, the COMMITTING pivot). A STAGING crash (txn=STAGING, CURRENT.json absent, manifest present) matches no row correctly. | §Decision 4e: table re-keyed on TXN-RECORD state (STAGING / COMMITTING / COMPLETED / ABORTED) as the primary discriminator, per BC-1.18.011 Invariant 3. The impossible "CURRENT.json status:staging" row is deleted. |
| H-2 | HIGH | Crash between step 8 (completed.json write) and gate→OPEN flip: gate=LOCKED, completed.json present, no live owner. Branch 2 ALREADY_MIGRATED path acquires no lock and performs no gate reconciliation; gate stays permanently LOCKED; all `.factory/specs/behavioral-contracts/` and `.factory/cycles/` mutations blocked forever. | §Decision 5c Branch 2: before exiting 0, MUST reconcile stale gate: acquire gate-state LOCK_EX; if gate ≠ OPEN and completed.json present with no active txn (txn absent, COMPLETED, or ABORTED): flip gate → OPEN; release gate lock; then exit 0 with `ALREADY_MIGRATED`. Fault-injection test mandate added. |
| H-3 | HIGH | Source/Origin and References cite `adv-cv-adr052-v13-closure-2026-09-13.md` which does NOT exist. No source for 4th–6th reviews cited accurately. Status finding-count (v1.5: "7 MEDIUM") mismatches the 4 mediums in the v1.5 Finding→Resolution table. | Source/Origin: replaced non-existent file citation with real provenance: local cascade D-1221 (pass-1), D-1222 (pass-2), D-1223 (pass-3); adv-local-adr052-pass3.md (written this burst by state-manager). Status finding-count corrected to 1C+4H+5M+2L. References cleaned of non-existent file. inputs[] + adv-local-adr052-pass3.md. |
| H-4 | HIGH | §Downstream Amendments 7/8/9 contained deferred-deferral placeholder text (the architect deferred providing the exact replacement text to a future dispatch). Product-owner cannot apply a placeholder; the text was a no-op instruction that blocked BC application. | §Downstream Amendments 7/8/9: placeholder removed; exact replacement text inlined, mirroring BC-1.18.011 v1.5 (CURRENT.json pointer swap as commit-point; intent log + matching-hash recovery; completed.json permanent terminal record; generation-first/canonical-fallback reader protocol; ADR-052 §Decision 7a/7b/7c citations). |
| M-1 | MED | Admission discriminator ambiguous across four sites: §5a PreToolUse describes reading only gate_state; §7a, BC-1.18.011 PC6(b), and error-taxonomy say block is driven by TXN-RECORD state regardless of PID; sites are inconsistent. | §5a: explicit text added: PreToolUse checks BOTH gate_state AND txn-record state; gate_state is the durable proxy that persists the txn-state block between per-event binary invocations; admit iff gate_state=OPEN AND no active txn (STAGING or COMMITTING); either condition alone triggers E-MAINTENANCE-001. |
| M-2 | MED | macOS `verify_and_exec_binary` Rust code sample omits the pre-execve mtime re-stat that §Decision 11 rationale item 4 mandates; prose-code gap from H-2(v1.4) closure. | §Decision 11 macOS code sample: mtime re-stat and comparison added immediately before `exec_by_pathname` call; code sample now matches the prose specification exactly. |
| M-3 | MED | Census/content abort error identifier triple-coded: step 3c uses "applicable error code"; §Error Code Semantics has CENSUS_MISMATCH_ABORT and CONTENT_PRESERVATION_ABORT as separate codes; trigger definitions reference the old (tautological) PC1 definition. Binary surfaces process exit codes, not HookResult. | Step 3c, §Error Code Semantics, and step 3b body aligned: CONTENT_PRESERVATION_ABORT = process exit code for PC1 (byte-for-byte concat SHA-256 mismatch against source_sha256) failures; CENSUS_MISMATCH_ABORT = process exit code for PC2 (ID-set exactly-one-shard violation) and E-SHD-005 (shard boundary) failures; binary is a process, not a hook; triggers updated to match corrected definitions. |
| M-4 | MED | Mechanism-A CLAUDE.md amendment scope uses `.factory/cycles/*/` wildcard, making POLICY-22 human-auditable scope non-literal; four exact config-driven target paths are known. | CLAUDE.md amendment text: `.factory/cycles/*/` wildcard replaced with the four exact append-log paths: `.factory/cycles/v1.0-brownfield-backfill/decision-log.md`, `.factory/cycles/v1.0-brownfield-backfill/burst-log.md`, `.factory/cycles/v1.0-brownfield-backfill/lessons.md`, `.factory/cycles/v1.0-brownfield-backfill/session-checkpoints.md`. |
| M-5 | MED | Stale-reservation cleanup runs only at recovery startup; a crashed agent's reservation causes spurious DRAIN_TIMEOUT_ABORT on a first-run migration (not a recovery). | §5a drain procedure: stale-PID reservation GC promoted to step 1 of EVERY drain (before DRAINING flip), not only recovery startup. |
| L-1 | LOW | H1 title contains "(v1.4 Fix-Burst)" version pin — stale; version lives in frontmatter only. | H1 title: "(v1.4 Fix-Burst)" stripped. |
| L-2 | LOW | "completed.json" and "completed.json" used interchangeably in prose and code; canonical path is lowercase `completed.json` per §Decision 7c step 8 which writes `.factory/migration-state/completed.json`. | All path-bearing occurrences aligned to lowercase `completed.json`. |

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

**The expiry check (authorization gate) occurs at §Decision 7c step 4, preceding the
fingerprint recheck (step 5) and the CURRENT.json pointer swap (step 6 — the single commit
point).** This location is (a) after all staging work is complete (so a timeout during staging
does not abort recoverable work) and (b) before any irreversible canonical-path visibility
change (the pointer swap at step 6). Note: the authorization gate and pointer swap are NOT
"immediately before" each other — step 5 (fingerprint recheck) intervenes between them.

**Authorization check logic:**
- If current UTC time is within `expires_after_hours` of `timestamp_utc`: pass; proceed to
  pointer swap.
- If expired AND txn record shows state=STAGING (pre-pivot, no canonical paths changed):
  ABORT cleanly — delete staging generation, update txn record → ABORTED, **flip gate →
  OPEN** (atomic write to gate_state file), release flock. The activation window elapsed
  before the pivot was reached. (H1 fix: gate MUST return to OPEN on every abort path.)
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

After migration reaches COMPLETED (completed.json written), the binary adds `consumed_at` to
the manifest (atomic overwrite). State-manager archives manifest to `.factory/migration-audit/`.

Re-run behavior is governed by TXN-RECORD state (primary discriminator per BC-1.18.011
Invariant 3) + completed.json presence. CURRENT.json is NOT the discriminator here:
it does not exist during STAGING (first written at step 6, the COMMITTING pivot) and
"CURRENT.json status:staging" is an impossible state.

| TXN-RECORD state | completed.json / manifest | Action |
|---|---|---|
| completed.json exists (terminal) | completed.json present | Exit 0 with `ALREADY_MIGRATED`; no lock needed; no manifest needed; **reconcile stale gate first** (§Decision 5c Branch 2 H-2 fix: acquire gate LOCK_EX; if gate≠OPEN and no active txn: flip→OPEN; release; then exit 0) |
| txn COMMITTING + valid manifest | CURRENT.json `status:committing` | Recovery: acquire flock; validate completion-only manifest or unexpired original; complete forward recovery |
| txn COMMITTING + manifest absent or expired | CURRENT.json `status:committing` | Abort with `RECOVERY_REQUIRES_REAUTHORIZATION`; human must issue completion-only manifest |
| txn STAGING + valid original manifest | CURRENT.json absent (pre-pivot) | Resume from staging: validate under exclusion; **RE-RUN full census against staged generation (EC-003 per H3 fix — step 3b must pass before proceeding)**; re-run authorization gate (step 4); proceed to fingerprint recheck (step 5) and pointer swap (step 6) |
| no txn record + manifest absent | CURRENT.json absent | Reject as unauthorized; exit non-zero |
| `--census` flag | any | Read-only; no txn record, no manifest, no flock required |

### Decision 5 — Dispatcher PreToolUse guard stack + native admission gate (v1.3 REDESIGNED)

#### 5a — Native maintenance admission gate: OPEN/DRAINING with writer reservations (v1.4 — closes F3, H1, H2)

**Dispatcher process model (v1.4 — closes H2):** The factory-dispatcher is a **per-event
binary** invoked once per PreToolUse or PostToolUse hook event and then exits. It is NOT a
persistent daemon. Consequently, an in-process `Mutex<{gate_state, active_writer_count}>`
cannot observe cross-process writer reservations — a PreToolUse binary increment and a
PostToolUse binary decrement are in SEPARATE PROCESSES. The admission mechanism MUST use
durable, file-system-level state for both gate_state and writer reservations.

**Gate states** (persisted to `.factory/migration-state/gate-state`, atomic replace-via-rename):
- `OPEN`: normal operation; new writer reservations accepted
- `DRAINING`: maintenance intent declared; no new writer reservations; coordinator awaiting quiescence
- `LOCKED`: coordinator has reached quiescence; migration in progress; no mutations admitted

**Writer reservation directory (v1.4 — replaces in-process counter, closes H2; v1.5 H-3 — tool_use_id correlation fix):**
`.factory/migration-state/reservations/` — each admitted writer creates a
`<tool_use_id>.reservation` file in this directory when admitted (PreToolUse), where
`tool_use_id` is the stable harness tool-invocation identifier shared across the PreToolUse
and PostToolUse hook pair for the same tool call. This allows the PostToolUse hook (a SEPARATE
PROCESS from PreToolUse, since the dispatcher is per-event) to identify and remove the exact
reservation file without any shared in-process state. A fresh random UUID is INCORRECT for
this purpose: PostToolUse cannot recompute a random value generated in a prior process, causing
permanent reservation leaks that prevent quiescence. The count of files in this directory is
the cross-process-durable `active_writer_count`. Quiescence = directory is empty.

**Atomic admission protocol (closes H2 atomicity gap; v1.6 M-1 — explicit dual-check):**
- **PreToolUse:** before admitting a mutation (Edit/Write/MultiEdit/Bash) targeting
  `.factory/specs/behavioral-contracts/` or `.factory/cycles/`:
  1. Acquire `LOCK_SH` flock on `.factory/migration-state/gate-state` file
  2. Read gate_state value
  3. Check for an active txn record: scan `.factory/migration-state/txn-*.json` for state IN
     (STAGING, COMMITTING)
  4. If gate_state = OPEN AND no active txn: create `reservations/<tool_use_id>.reservation`
     file; release `LOCK_SH`; proceed (admit)
  5. If gate_state ≠ OPEN (DRAINING or LOCKED) OR active txn exists (STAGING or COMMITTING):
     release `LOCK_SH`; return E-MAINTENANCE-001 immediately (block)
  **Invariant (M-1 — dual-check):** Both conditions must be satisfied for admission: gate_state
  MUST be OPEN AND no active txn may exist. `gate_state` is the durable cross-binary proxy that
  persists txn blocking between per-event dispatcher invocations (the gate is set to LOCKED when
  the txn is STAGING or COMMITTING). Checking both guards against the race window between gate
  flip and txn record write. The txn record check is the authoritative "is maintenance in progress"
  signal per §Decision 7a ordinary-writer blocking; gate_state is the fast-path durable proxy.
  The shared lock in step 1 blocks ONLY while the coordinator holds `LOCK_EX` to flip gate state.
  Between gate-state flips, parallel admissions proceed without blocking each other.

- **PostToolUse:** after tool completion (success or failure): remove the reservation file
  `reservations/<tool_use_id>.reservation` created by this invocation's PreToolUse hook, using
  the same `tool_use_id` the harness provides to both hook events for the same tool invocation.
  If the reservation file does not exist (writer crashed between Pre and Post): no-op (normal).

**Bash admission and reservation (v1.5 — closes M-6):** Every admitted Bash command whose
write effect targets `.factory/specs/behavioral-contracts/` or `.factory/cycles/` creates a
`<tool_use_id>.reservation` file by the same mechanism as Edit/Write/MultiEdit. The §5c
conservative-Bash-admission classifier determines whether a Bash command has a write effect on
those paths; commands the classifier determines to have NO write effect (read-only forms such as
ls, cat, grep, git log, git status, git diff) do not require a reservation and are not subject
to the gate admission protocol above. Quiescence waits for ALL in-flight reservations,
including Bash ones.

**Drain procedure (executed by maintenance coordinator before building staging generation):**
1. **Stale-PID reservation GC (v1.6 M-5 — runs at start of EVERY drain, not only recovery):**
   Scan `reservations/` directory for files whose creating PID is no longer alive (file content
   = PID). Remove stale files. This prevents a crashed writer from causing spurious
   DRAIN_TIMEOUT_ABORT on a first-run migration. This cleanup runs here (before DRAINING flip)
   so it does not race with the quiescence check in step 3.
2. Acquire advisory flock on `exclusive.lock` (§Decision 7a)
3. Acquire `LOCK_EX` flock on `gate-state` file; write gate_state = DRAINING; release `LOCK_EX`
   (LOCK_EX blocks new PreToolUse admissions while the flip is in progress)
4. Poll `reservations/` directory until empty; timeout at 30s → **ABORT: flip gate → OPEN
   (under LOCK_EX), release both flocks.** (H1 fix: abort path MUST flip gate → OPEN)
   (v1.5 L-4 LIVENESS NOTE: The 30s drain timeout is a safety net, not a production
   operating assumption. F4 activation SHOULD be scheduled when the system is quiescent with
   no concurrent agent Edit/Write/MultiEdit/Bash operations; the operator is responsible for
   choosing an activation window with no concurrent agent activity to avoid spurious
   DRAIN_TIMEOUT_ABORT.)
5. Snapshot now-quiescent inputs (compute `source_sha256` of all source files; also capture
   the original BC-INDEX.md body ID census — the complete set of `BC-X.YY.NNN` IDs — and
   store in txn record for use by step 3b PC2)
6. Acquire `LOCK_EX`; write gate_state = LOCKED; release `LOCK_EX`
7. Write txn record with state=STAGING (§Decision 7a)

**COMPLETED release:** When completed.json written (step 8), flip gate `LOCKED → OPEN` (under
`LOCK_EX` on gate-state file). This MUST succeed before the flock is released.

**Abort gate-reset obligation (v1.4 — closes H1):** EVERY abort and rollback path MUST flip
gate → OPEN before returning. This applies to:
- Drain-timeout abort (step 3 above)
- Expiry abort at §7c step 4 (authorization gate check)
- Fingerprint abort at §7c step 5
- Census abort at §7c step 3b (new in v1.4)
- Any other early-exit path that sets txn → ABORTED

**Crash-recovery gate reconciliation (v1.4 — closes H1):** A recovery process that acquires
the flock on `exclusive.lock` and finds gate_state = LOCKED or DRAINING MUST:
- Check for an active txn record (`txn-*.json` with state STAGING or COMMITTING)
- If NO active txn exists (txn absent, COMPLETED, or ABORTED): flip gate → OPEN; proceed
- If active txn exists (STAGING or COMMITTING): gate is legitimately LOCKED; proceed with
  forward recovery per §Decision 4e and §Decision 7b

**Stale reservation cleanup:** Stale-PID reservation GC runs at the START OF EVERY DRAIN
(drain procedure step 1, above) — not only at recovery startup. This ensures a crashed
writer's reservation does not cause spurious DRAIN_TIMEOUT_ABORT on a first-run migration.
On crash recovery startup (separate from a first-run drain), the same GC also runs before
re-attempting the drain.

**Fault-injection test mandate (v1.4 — H1; v1.5 — H-3, M-6; v1.6 — M-5):** Tests MUST verify:
- Drain-timeout abort: gate returns to OPEN; next PreToolUse is admitted
- Expiry abort at step 4: gate returns to OPEN; txn = ABORTED
- Fingerprint abort at step 5: gate returns to OPEN; txn = ABORTED
- Census abort at step 3b: gate returns to OPEN; txn = ABORTED
- Writer admitted just before DRAINING flip: coordinator waits for its PostToolUse before
  proceeding past step 3 (quiescence wait)
- (H-3) PreToolUse creates `reservations/<tool_use_id>.reservation`; PostToolUse (separate
  process, same tool_use_id provided by harness) removes it; directory is empty after both
  hooks fire for the same tool invocation
- (H-3) Stale-PID cleanup: a reservation file whose PID is no longer alive is removed by the
  recovery startup scan; subsequent drain correctly proceeds to quiescence
- (M-6) Long-running admitted Bash with write effect creates a reservation; coordinator waits
  for PostToolUse reservation removal before proceeding past step 4 of drain procedure
- (M-5) A crashed writer's stale reservation is cleaned up by drain step 1 GC on a FIRST-RUN
  migration (not only on recovery); subsequent drain reaches quiescence and proceeds normally

**Fingerprint recheck:** Retained at §Decision 7c step 5 as defense-in-depth after quiescence.
Not the drain mechanism.

**Non-upgrade:** Do NOT use a shared→exclusive flock upgrade as the drain mechanism. Use the
explicit OPEN→DRAINING→LOCKED state machine with the gate-state file as described above.

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

**Branch 2 — Terminal-state no-op (closes F8 COMPLETED-rerun gap; v1.6 H-2 — gate reconciliation):**
- Trigger: `completed.json` exists at `.factory/migration-state/completed.json`
- No migration manifest required; no exclusive.lock flock required
- **Gate reconciliation before exit (H-2 fix):** Before exiting 0, the guard MUST check and
  reconcile a stale gate to prevent a permanent LOCKED state from a crash between step 8
  (completed.json write) and the COMPLETED gate→OPEN flip:
  1. Acquire `LOCK_EX` on gate-state file
  2. Read gate_state value
  3. If gate_state ≠ OPEN AND completed.json present AND no active txn record exists
     (txn absent, or txn state = COMPLETED or ABORTED): write gate_state = OPEN (atomic)
  4. Release `LOCK_EX`
  This reconciliation is a no-op when the gate is already OPEN (normal case). It is also a
  no-op when an active STAGING or COMMITTING txn exists (gate is legitimately LOCKED).
- Binary exits 0 with `ALREADY_MIGRATED`
- **Fault-injection test mandate (H-2):** A test MUST simulate the crash window between
  completed.json write and gate→OPEN flip. Verify that a subsequent Branch 2 invocation
  reconciles gate → OPEN; verify that subsequent Edit/Write PreToolUse admissions are
  accepted after this reconciliation.

**Branch 3 — New activation:**
- Trigger: no `completed.json` + no CURRENT.json with `status: committing`
- Full validation: exact command string match; metacharacter rejection; alternate-path rejection;
  manifest content validation (parseable, `migration_id` matches, not expired, `activation_id`
  is valid UUID); executable digest verification (§Decision 11)
- If any check fails: REJECT with descriptive error

**Branch 4 — Recovery:**
- Trigger: CURRENT.json `status: committing` exists
- Full validation: command string match; metacharacter rejection; executable digest verification;
  completion-only manifest validation (§Decision 4d) OR unexpired original manifest
- If any check fails: REJECT

**Conservative Bash admission for all other commands (closes F7, H4):**
When the txn record exists with state=STAGING or COMMITTING, ANY Bash command not matching
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
- Any Bash command with `>` targeting `.factory/specs/behavioral-contracts/` while txn=STAGING or COMMITTING → REJECTED (F7/H4 test)
- `{root}/target/release/factory-dispatcher migrate-bc-index` (when completed.json exists) → BRANCH-2 pass
- Abort during DRAINING (drain-timeout) → gate returns to OPEN; next PreToolUse mutation is admitted (H1 test)

### Decision 6 — Audit trail: durable, tamper-evident record (NIST AU-9) (v1.5 H-1: inlined; completed.json replaces COMMITTED phase marker per v1.3 redesign)

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

2. **Durable audit record (NIST AU-9):** Immediately after the migration reaches COMPLETED
   state (completed.json written at §Decision 7c step 8), state-manager commits to
   factory-artifacts:
   - The captured census stdout as `migration-audit/migrate-bc-index-YYYY-MM-DD-census.txt`
     (or `backfill-append-logs-...`)
   - The armed-activation manifest (consumed copy from `.factory/activation/`)
   - The completed.json terminal record
   - The migration binary's `--version` output

   The factory-artifacts commit hash is recorded in STATE.md's burst-log for the activation
   burst. This factory-artifacts commit is the tamper-evident record: it is immutable in git
   history and not subject to session compaction.

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
- If `result == EWOULDBLOCK`: another process holds the lock → return E-MAINTENANCE-001 immediately.
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

**Recovery-owner claim protocol (closes F5 exclusive takeover gap; M1 — fencing_generation
is AUDIT-ONLY):**
A recovery process that acquires the flock after detecting a stale owner (txn record exists
with state=STAGING or COMMITTING) MUST claim the transaction by:
1. Reading the current `fencing_generation` from the txn record
2. Incrementing it by 1 (bump = claim; monotonic counter, never decremented)
3. Writing the updated txn record with the new `fencing_generation` and its own PID in the
   lock file body — under the held flock
4. Every subsequent write step by the recovery owner records the current `fencing_generation`
   in its write metadata — **AUDIT-ONLY, for traceability** (crash-restart PID-reuse
   detection, not an enforcement gate). The advisory flock provides actual mutual exclusion:
   only one process can hold `LOCK_EX` on `exclusive.lock` at a time; a stale recovery
   process cannot hold the flock and a live recovery process simultaneously. No resource
   rejects a write based on fencing_generation value alone.

**Ordinary-writer blocking (closes F5 regardless-of-PID-liveness gap):**
The native admission gate (§Decision 5a) blocks ALL mutation tool calls targeting
`.factory/specs/behavioral-contracts/` or `.factory/cycles/` whenever a txn record exists at
`.factory/migration-state/txn-*.json` with state IN (STAGING, COMMITTING). This check is
independent of flock state and PID liveness. A txn record in STAGING or COMMITTING state means
the migration has exclusive rights to those paths, even if the flock-holding process is
temporarily absent (crash recovery scenario). Writers receive E-MAINTENANCE-001.

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
- This directory is CONTENT-IMMUTABLE after all content is written and synced (step 2):
  file contents are never modified. Files may be moved (renamed) to canonical paths during
  step 7; a moved file is no longer accessible at its `gen-<uuid>/` path (its content is at
  the canonical path instead). The reader's canonical-first fallback (see Reader Protocol
  below) ensures the new content is always accessible at one of the two paths. (C1 fix)

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

3b. **Pre-pivot content-preservation + census gate (v1.4 — closes H3; BC-1.18.011 PC1/PC2;
    v1.6 C-1 — rewrites PC1 and PC2 to remove tautology).**
    Before the authorization gate, verify the staged generation is correct and complete.
    This step MUST execute before the irreversible CURRENT.json pointer swap at step 6.

    - **Staging-file integrity (distinct from PC1):** For each staged file, verify
      `sha256(staged_file) == expected_post_hash` (from the INTENT record written in step 3).
      This confirms no file was silently modified after the staging sync in step 2. On any
      mismatch → step 3c (abort) with `CONTENT_PRESERVATION_ABORT`.
      **NOTE (v1.6 C-1):** This check is a STAGING INTEGRITY guard, NOT content-preservation
      (PC1). It compares a staged file's hash against its own intent-log entry — a necessary
      but trivially-satisfiable check (sha256(x)==sha256(x) when the intent log was written
      from the same file). It does not verify the split is semantically correct. PC1 and PC2
      below are the real correctness gates.

    - **Content-preservation (PC1 — BC-1.18.011 PC1):** Reconstruct and verify byte-for-byte:
      concatenate all staged shard files in `SS-01`..`SS-10` order plus the staged retained lean
      `BC-INDEX.md` body (the new header-only form per BC-1.18.010 Invariant 3). Compute
      SHA-256 of this concatenation. Compare against `source_sha256` in the txn record (the
      SHA-256 of the ORIGINAL pre-split `BC-INDEX.md` body captured at the quiescence snapshot
      in §Decision 5a drain step 4). On any mismatch → step 3c (abort) with
      `CONTENT_PRESERVATION_ABORT`. This is the byte-for-byte fidelity gate required by
      BC-1.18.011 PC1: no row content may be altered, dropped, or reordered during the split.

    - **Independent census (PC2 — BC-1.18.011 PC2):** Perform a fresh enumeration of ALL
      `BC-X.YY.NNN` IDs in the ORIGINAL `BC-INDEX.md` body (the ID set captured at quiescence
      and stored in the txn record). For each ID in this original-census set, verify:
      (a) the ID appears in EXACTLY ONE staged shard file (`gen-<uuid>/shards/BC-INDEX-SS-NN.md`
          or a sub-shard);
      (b) the ID does NOT appear in the staged lean `BC-INDEX.md` retained body (zero
          occurrences, per BC-1.18.010 Invariant 3).
      Verify the union count of shard IDs equals the original-census count exactly.
      **EC-001 MUST abort:** A byte-identical concatenation that nonetheless duplicates one ID
      in two shards while dropping another ID with compensating count — so that total count
      matches but the ID SET does not — is a FAILING migration (BC-1.18.011 Invariant 2: "no BC
      row is ever counted twice or dropped"). Count comparison alone cannot detect EC-001; the
      ID-set check is mandatory. On any violation → step 3c (abort) with `CENSUS_MISMATCH_ABORT`.

    - **Shard boundary invariants (E-SHD-005):** Verify each sub-shard file count is within
      shard capacity bounds. On violation → step 3c (abort) with `CENSUS_MISMATCH_ABORT`.

3c. **Gate failure abort (on any step 3b check failure; v1.6 M-3 — exit code aligned):**
    - Delete staging generation directory `gen-<uuid>/`
    - Update txn record: state → ABORTED
    - Flip gate → OPEN (atomic write to gate_state file, under `LOCK_EX`) (H1 fix)
    - Release flock on `exclusive.lock`
    - Exit non-zero with the applicable process exit code:
      - `CONTENT_PRESERVATION_ABORT` (exit 2): staging-file integrity failure (sha256 mismatch
        vs intent-log expected_post_hash) OR PC1 failure (byte-for-byte concat SHA-256 mismatch
        vs source_sha256)
      - `CENSUS_MISMATCH_ABORT` (exit 2): PC2 failure (ID-set exactly-one-shard violation,
        including EC-001 dup+drop case) OR E-SHD-005 shard boundary violation
    These are PROCESS EXIT CODES emitted by the migration binary, not HookResult values.
    See §Error Code Semantics for full trigger, severity, and exit behavior.

4. **Authorization gate check (§Decision 4d).** Verify expiry. If expired and state=STAGING:
   ABORT cleanly (update txn → ABORTED, **flip gate → OPEN** (H1 fix), release flock).
   If passes: proceed.

5. **Fingerprint recheck (defense-in-depth).** Re-read all source files, recompute SHA-256.
   Compare against `source_sha256` in txn record (captured at quiescence). If any differ:
   ABORT; delete staging generation; txn record → ABORTED; **flip gate → OPEN** (H1 fix);
   release flock; exit with FINGERPRINT_MISMATCH_ABORT. Require re-activation.

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

**Reader protocol (v1.5 C-1 — generation-first / canonical-fallback; replaces v1.4 canonical-first which was a regression for in-place-overwrite targets):**
Readers that access BC-INDEX paths DURING the migration window MUST:
1. Check `.factory/migration-state/completed.json` — if exists: migration complete; canonical
   paths are current and authoritative. Use canonical paths.
2. Check `.factory/migration-state/CURRENT.json` — if `status: committing`: migration is in
   progress. For each required file:
   a. **Try `gen-<generation_id>/` path FIRST.** If the file exists at its generation-dir
      path, use it. (File present in gen ⟹ not yet moved to canonical ⟹ new-generation
      content. This is the correct path for both net-new shard files and BC-INDEX.md before
      its rename in step 7.)
   b. **Fall back to canonical path** if the generation-dir path is absent. (File absent from
      gen ⟹ already renamed to canonical by step 7 progress ⟹ canonical now holds
      new-generation content, guaranteed by rename(2) atomicity.)
   This generation-first / canonical-fallback protocol is correct for BOTH net-new shard files
   (canonical doesn't exist until created by step 7) AND in-place-overwrite targets such as
   BC-INDEX.md (canonical holds OLD monolithic body until step 7's specific rename for that
   file; canonical-first — the v1.4 protocol — returns stale content before that rename).
   ENOENT is impossible for any new-generation file: a file is at gen-uuid/ OR at canonical,
   never at neither, because `rename(2)` is atomic on POSIX.
3. If neither file exists: migration not started; use BC-INDEX.md (legacy form).
This protocol is unambiguous in all states including after crash recovery and cleanup.

**Fault-injection reader test mandate (v1.5 — C-1):** A test MUST read BC-INDEX.md at every
point of step-7 progress while CURRENT.json `status=committing` and assert the LEAN (new)
body is always observed — never the monolithic body. This test must cover: (a) before
gen-uuid/BC-INDEX.md is renamed (generation-first returns new lean body from gen-uuid/); (b)
after gen-uuid/BC-INDEX.md is renamed (canonical fallback returns new lean body from canonical
path). The stale monolithic body MUST never be returned at any step-7 progress point.

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

#### Skipped-control inventory (v1.5 H-1: inlined from v1.2; unchanged in substance)

| Skipped control | What it actually enforces | Equivalent enforcement in migration binary or ratified waiver |
|-----------------|---------------------------|---------------------------------------------------------------|
| `brownfield-discipline` PreToolUse (Edit\|Write\|MultiEdit) | **Protects `.reference/` from governed writes** (script lines 4-9 and 34-38 check write targets against the `.reference/` path prefix) | Migration binary does not write to `.reference/`; all targets are explicitly enumerated in §Decision 8 allowed-paths list. **No `.reference/` writes possible by construction.** |
| `factory-branch-guard` PreToolUse (Edit\|Write\|MultiEdit) | **Checks that the active worktree is NOT the factory-artifacts branch before permitting a mutation** (script lines 67-82 verify worktree identity and branch name) | Migration binary operates on the main working tree, not the factory-artifacts worktree. The migration targets (shard files, BC-INDEX.md) are in the main worktree. This guard does not apply to the migration binary's execution context. **EXPLICIT WAIVER (POLICY 22 authorized):** migration is always run in the main worktree; the guard's branch-check intent is satisfied because main-worktree BC mutations are exactly what this migration is authorized to perform. |
| `validate-factory-path-staging` PreToolUse (Bash) | Validates `.factory/` writes via Bash use proper channels | Guard amended (§Decision 5b) with full-command pre-shell classifier and manifest content validation (§5c); the amendment IS the equivalent enforcement |
| `validate-factory-path-staged` PostToolUse (Bash) | **IS registered for PostToolUse Bash** (hooks-registry.toml entry for PostToolUse Bash); verifies `.factory/` mutations from Bash are properly staged for factory-artifacts | **MUST be amended** by devops-engineer to recognize the governed migration subcommands (same absolute-path-pinned forms as §Decision 3) and pass them through without requiring Edit/Write staging semantics. Post-migration, devops-engineer performs the factory-artifacts commit manually as the audit record (§Decision 6). Guard amendment is a cluster-5 activation deliverable. |
| `validate-count-propagation.sh` PostToolUse behavior | Verifies count propagation across BC-INDEX | Migration binary's independent census cross-check against `total_bcs` oracle provides equivalent verification before any write; count propagation is a post-migration steady-state concern |
| POL-3 "NEVER use Python/sed/echo bypass" | Prevents unstructured, unvalidated `.factory/` writes | **EXPLICIT WAIVER (POLICY 22 authorized):** The migration binary is VSDD-authored, TDD-covered (VP-132/VP-133/VP-134), adversarially-reviewed. It implements the full BC-1.18.008/BC-1.18.011 specification. This waiver is narrowly scoped to the two governed migration subcommands and expires when the migration completes. |

#### Single authoritative allowed-write-targets list (v1.4 — closes F10, C2)

The exception covers writes to the following paths ONLY. This list is the SINGLE source of
truth for both the binary's containment checks AND the CLAUDE.md amendment text below:

```
# B2 migration targets (BC-INDEX body-split) — v1.4 NORMALIZED (closes C2)
.factory/specs/behavioral-contracts/BC-INDEX.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.a.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.b.md
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.c.md
  (and any additional single-lowercase-letter suffix in [a-z]; validate_write_target()
   accepts the pattern BC-INDEX-SS-<NN>.<letter>.md for any letter in [a-z])
.factory/specs/behavioral-contracts/shards/BC-INDEX.shard-manifest.toml
.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.manifest.toml
  (generic pattern; covers BC-INDEX-SS-05.manifest.toml, BC-INDEX-SS-06.manifest.toml,
   and any other subsystem sub-manifest)

# A migration targets (four exact config-specified append-log files — v1.6 M-4)
.factory/cycles/v1.0-brownfield-backfill/decision-log.md
.factory/cycles/v1.0-brownfield-backfill/burst-log.md
.factory/cycles/v1.0-brownfield-backfill/lessons.md
.factory/cycles/v1.0-brownfield-backfill/session-checkpoints.md
  (validate_write_target() MUST verify the exact path matches one of these four paths, not
   merely the `.factory/cycles/` prefix; the wildcard form is NOT an accepted pattern)

# Migration operational state (both migrations)
.factory/migration-state/
.factory/migration-state/gen-<uuid>/

# Activation and audit (both migrations)
.factory/activation/
.factory/migration-audit/
```

Where `<NN>` is a two-digit subsystem number, `<letter>` is any single lowercase ASCII letter
in `[a-z]`, and `<uuid>` is the runtime generation UUID.

**Sub-shard naming rationale (v1.4 — closes C2):** The v1.3 allowlist used hyphen-uppercase
suffixes (`-A.md`, `-B.md`) which diverged from the canonical naming established in ADR-051
§Decision 7 (`BC-INDEX-SS-NN.a.md`, `.b.md`, `.c.md`) and referenced in BC-1.18.011 EC-004.
`validate_write_target()` MUST implement the pattern check as a programmatic guard, not a
hardcoded list of specific letters, to support SS-05's three-way sub-split (`.a`, `.b`, `.c`)
and any future subsystem sub-sharding without requiring an allowlist update.

**Ratification-time test mandate (v1.4 — closes C2):** The cluster-5 TDD test suite MUST
include a test asserting that every path referenced in BC-1.18.010 and BC-1.18.011 is accepted
by `validate_write_target()`. At minimum:
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.a.md` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.b.md` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.c.md` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-06.a.md` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05.manifest.toml` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-06.manifest.toml` → ACCEPTED
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05-A.md` → REJECTED (hyphen-upper format)
- `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-05-B.md` → REJECTED (hyphen-upper format)
- `.factory/specs/behavioral-contracts/BC-INDEX.md` → ACCEPTED (v1.5 L-2: the in-place overwrite target itself must pass)
- `.factory/specs/behavioral-contracts/shards/BC-INDEX.shard-manifest.toml` → ACCEPTED (v1.5 L-2: top-level shard manifest)
This test MUST pass before cluster-5 TDD proceeds to implementation.

**Containment check (required before every write):** Before any write to any target path, the
migration binary MUST verify the resolved canonical path prefix matches one of the above entries.
Any target that does not match MUST be rejected with a non-zero exit regardless of manifest state.
This check is implemented in a `validate_write_target(path: &Path) -> Result<(), MigrationError>`
function called by every path that produces a filesystem write.

#### CLAUDE.md amendment text (v1.4 — generated from the same allowlist above)

The human MUST apply the following amendment to `CLAUDE.md` as part of POLICY 22 ratification.

**Location:** `## Conventions (Code-Level)` section, `### Forbidden patterns` table, the row
for TD-FACTORY-HOOK-BYPASS-001 P0.

**Amendment (generated from the authoritative allowlist above — no scope expansion):**

```
ADR-052 EXCEPTION (POLICY 22 ratified): The governed one-time shard migration binary
(`{project-root}/target/release/factory-dispatcher migrate-bc-index` and
`backfill-append-logs`) may write to the following paths ONLY:
  `.factory/specs/behavioral-contracts/BC-INDEX.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.a.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.b.md`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.c.md`
  (and any additional dot-lowercase letter suffix [a-z] — validated by validate_write_target())
  `.factory/specs/behavioral-contracts/shards/BC-INDEX.shard-manifest.toml`
  `.factory/specs/behavioral-contracts/shards/BC-INDEX-SS-<NN>.manifest.toml`
  (generic; covers all subsystem sub-manifests including SS-05 and SS-06)
  `.factory/cycles/v1.0-brownfield-backfill/decision-log.md`
  `.factory/cycles/v1.0-brownfield-backfill/burst-log.md`
  `.factory/cycles/v1.0-brownfield-backfill/lessons.md`
  `.factory/cycles/v1.0-brownfield-backfill/session-checkpoints.md`
  (these four are the config-specified append-log targets for mechanism A; validate_write_target()
   verifies the exact path matches one of the above four, not merely the cycle prefix)
  `.factory/migration-state/` (flock inode, txn record, intent log, CURRENT.json, completed.json,
                                gate-state, reservation dir, staging generation)
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

POST-SUCCESS OBLIGATIONS (after completed.json written):
(f) Durable factory-artifacts commit records census stdout, activation manifest, completed.json,
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
// DEPENDENCY: /proc must be mounted. Fails closed if /proc unavailable (container, chroot).
// In environments without /proc, the fallback errors; only execveat AT_EMPTY_PATH is available.
execveat(fd, CStr::from_bytes_with_nul(b"\0")?, argv, envp, AT_EMPTY_PATH)?;
```

`AT_EMPTY_PATH` on Linux 3.19+ (glibc 2.34 wrapper) executes the file referred to by `fd`
without re-resolving a pathname. The binary being executed is byte-for-byte identical to the
one that was hashed.

**O_PATH caveat:** Do NOT open with `O_PATH` for this purpose — `O_PATH` fds cannot be `read()`
through, so hashing is not possible. Use `O_RDONLY`.

#### macOS/darwin-arm64 path (primary operator platform — documented residual TOCTOU — NEEDS HUMAN SIGN-OFF at POLICY 22)

`fexecve` and `execveat AT_EMPTY_PATH` are **unavailable on macOS** (darwin-arm64). Apple's
documented exec interfaces (`execl`, `execle`, `execlp`, `execv`, `execvp`, `execvP`, `execve`)
are all pathname-based. No Apple-documented fd-binding exec primitive exists. The strongest
available evidence is Apple's exec man-page set exposing only pathname-based APIs; corroborating
non-Apple SDK/source reports confirm the missing symbol. This is not backed by an explicit
Apple negative statement, but confidence is HIGH.

**Architect decision for macOS/darwin-arm64 (PRIMARY OPERATOR PLATFORM — v1.4 M5 reframe):
freeze build under maintenance lock + programmatic mtime guard + accept documented residual
TOCTOU window (requires POLICY 22 human acknowledgment).**

macOS/darwin-arm64 is the primary operator runtime. The residual exec-TOCTOU window is the
DEFAULT runtime condition for this product, not a secondary concern. This section documents
the production-grade mitigations and the residual risk that requires explicit human sign-off.

Rationale:
1. macOS/darwin-arm64 is the primary operator platform. The risk-framing of the residual TOCTOU
   window MUST reflect that this is the default runtime, not a fallback.
2. The maintenance lock (advisory flock) is held before the digest check. No agent edit/write
   paths can replace the binary (they target `.factory/`, not `target/release/`).
3. The realistic threat is a concurrent `cargo build` replacing the binary.
4. **Programmatic mtime guard (v1.4 M5; v1.5 H-2 corrected: re-stat immediately before execve):**
   Record `mtime(target/release/factory-dispatcher)` immediately after the digest check.
   IMMEDIATELY BEFORE the `execve` pathname call, re-stat the binary's mtime and compare
   against the recorded value. If mtime changed between the initial digest check and the
   pre-exec re-stat: abort with `E-BINARY-INTEGRITY-FAILURE` BEFORE the exec call. Do NOT
   re-hash through a fresh fd on this path (to minimize the window between re-stat and execve);
   mtime change is sufficient signal to abort.
   **Correction from v1.4:** the previous wording said "after exec returns" — a successful
   `execve` NEVER returns (it replaces the calling process), so the post-exec check fired only
   on exec failure, leaving the dangerous successful-substitution-then-exec case completely
   undetected. Moving the re-stat to immediately before the `execve` syscall closes this gap:
   the detectable window shrinks from (digest-check → exec) to (pre-exec-re-stat → execve),
   which is sub-CPU-instruction under normal OS scheduling.
   This does NOT cryptographically close the TOCTOU window, but it narrows the undetectable
   surface to near-zero under a quiescent system with no concurrent cargo build.
5. **Hard pre-flight checklist item (v1.4 M5):** The activation procedure MUST require as a
   HARD PREREQUISITE (not a soft recommendation) before running the migration: "Verify no
   cargo build process is running: `pgrep -x cargo-build` returns empty." This item MUST
   appear in the human-visible activation checklist at the F4 POLICY 22 gate.
6. Protected staging (copy to `/tmp/` + `UF_IMMUTABLE`) does not provide meaningful security:
   `UF_IMMUTABLE` is owner-changeable, `/tmp/` is owner-writable, and the staging step has
   its own TOCTOU window.
7. The residual window is intra-process (sub-millisecond under quiescent system) with no
   concurrent build running and the mtime guard active.

**macOS implementation (v1.6 M-2 — mtime re-stat added immediately before exec):**
```rust
#[cfg(target_os = "macos")]
fn verify_and_exec_binary(binary_path: &Path, ...) -> Result<(), ExecError> {
    let fd = File::open(binary_path)?;
    let computed = sha256_read_through_fd(&fd)?;
    if computed != stored_digest {
        return Err(ExecError::DigestMismatch);
    }
    // Record mtime immediately after digest check for pre-exec comparison.
    let mtime_after_digest = stat_mtime(binary_path)?;

    // No fd-binding exec available on macOS; execute by pathname under held maintenance lock
    // DOCUMENTED RESIDUAL TOCTOU: a concurrent cargo build between hash and exec could
    // substitute different bytes. Mitigation: no concurrent build during F4 activation.
    // This residual risk requires human acknowledgment at ratification (see §Decision 11).

    // Programmatic mtime guard: re-stat IMMEDIATELY BEFORE execve syscall.
    // If mtime changed between digest check and this re-stat, a substitution may have
    // occurred — abort BEFORE executing. This narrows the undetectable window from
    // (digest-check → exec) to (pre-exec-re-stat → execve), which is sub-instruction
    // under a quiescent system. Does NOT close the TOCTOU window cryptographically.
    let mtime_pre_exec = stat_mtime(binary_path)?;
    if mtime_pre_exec != mtime_after_digest {
        return Err(ExecError::BinaryIntegrityFailure);
    }

    exec_by_pathname(binary_path, ...)?
}
```

**Test:** A test MUST verify that digest mismatch (binary replaced between open and exec check)
causes the migration to abort before exec with `E-BINARY-INTEGRITY-FAILURE` rather than silently
executing the replacement. On Linux, this test also verifies the execveat AT_EMPTY_PATH path via
a test double that intercepts the execveat call and confirms the fd argument matches the opened fd.

**Human sign-off required (v1.5 H-2 — corrected mtime-guard semantics):** The macOS/darwin-arm64
(primary operator platform) residual TOCTOU window is the sub-instruction gap between the
pre-exec mtime re-stat and the `execve` pathname call — narrowed from the v1.4 framing of
(digest-check → exec). The corrected mtime guard (§Decision 11 rationale item 4) detects
substitution that occurs BEFORE the pre-exec re-stat; it does NOT detect substitution in the
remaining sub-instruction window between re-stat and execve. The human MUST explicitly
acknowledge this corrected residual risk as part of POLICY 22 ratification. The migration
activation procedure MUST include as a HARD PREREQUISITE checklist item: "Verify no cargo
build process is running (`pgrep -x cargo-build` returns empty)." This acknowledgment must
be recorded in the D-NNN entry that ratifies ADR-052.

---

## Error Code Semantics

The following error codes are emitted by the migration binary. The product-owner MUST add
catalog rows to `error-taxonomy.md` for each code. This section defines the trigger, severity,
and exit behavior to drive that addition.

| Error Code | Trigger | Severity | Exit |
|---|---|---|---|
| `E-BINARY-INTEGRITY-FAILURE` | SHA-256 digest of the binary (computed through opened fd or path) does not match the stored/expected digest; OR programmatic mtime guard detects mtime change between digest-check and exec attempt on macOS | BLOCKED — migration aborted before exec | exit 2 |
| `RECOVERY_REQUIRES_REAUTHORIZATION` | Recovery process encounters a COMMITTING transaction whose original manifest has expired AND no completion-only recovery manifest is present or valid | BLOCKED — cannot forward-recover without fresh authorization | exit 2 |
| `EXPIRY_ABORT` | Activation manifest has expired (timestamp_utc + expires_after_hours < now) AND txn record state is STAGING (pre-pivot); clean abort | NON-ERROR termination — no canonical paths changed; re-activation required | exit 1 |
| `FINGERPRINT_MISMATCH_ABORT` | Source files changed between quiescence snapshot and fingerprint recheck at step 5 (sha256 of source != source_sha256 in txn record) | BLOCKED — abort before irreversible step | exit 2 |
| `DRAIN_TIMEOUT_ABORT` | Coordinator waited 30 seconds for active writer reservations to drain to zero; quiescence not achieved | BLOCKED — migration deferred; gate returned to OPEN | exit 2 |
| `ARCH_INDEX_PARITY_ABORT` | Three-way ARCH-INDEX parity check fails: config.arch_index_sha != manifest.approved_arch_index_sha OR != live ARCH-INDEX SHA | BLOCKED — abort before any staging work | exit 2 |
| `COMPLETION_MANIFEST_REJECTION` | Completion-only recovery manifest fails validation: activation_id mismatch, staged_generation_id mismatch, fencing_generation mismatch, or allowed_steps includes pre-pivot steps | BLOCKED — abort recovery; require new completion-only manifest | exit 2 |
| `CENSUS_MISMATCH_ABORT` | Pre-pivot census (step 3b PC2): any `BC-X.YY.NNN` ID from the original-census set appears in zero or more than one staged shard file (ID-set exactly-one-shard violation; detects EC-001 dup+drop even when total count is equal); OR E-SHD-005 shard boundary violation. Process exit code. | BLOCKED — abort before pointer swap; txn → ABORTED; gate → OPEN | exit 2 |
| `CONTENT_PRESERVATION_ABORT` | Pre-pivot checks (step 3b): staging-file integrity failure (sha256(staged_file) != expected_post_hash for any staged file, indicating modification after staging sync); OR PC1 byte-for-byte content-preservation failure (SHA-256 of reconstructed shard concatenation + retained lean body does not match source_sha256 from txn record). Process exit code. | BLOCKED — abort before pointer swap; txn → ABORTED; gate → OPEN | exit 2 |
| `ALREADY_MIGRATED` | `completed.json` exists at `.factory/migration-state/completed.json` | NON-ERROR sentinel — migration previously completed; no action taken | exit 0 |
| `E-MAINTENANCE-001` | Mutation tool call attempted while gate state is DRAINING or LOCKED, OR while txn record exists with state STAGING or COMMITTING | BLOCKED — writer must retry after migration completes | exit 2 (guard layer) |

**Note for product-owner:** Add one catalog row per code to `error-taxonomy.md`. The `exit 0`
for `ALREADY_MIGRATED` is a deliberate sentinel — callers that re-invoke the migration binary
after completion MUST not treat this as a failure. The `exit 1` for `EXPIRY_ABORT` signals
"no harm done, but re-activation required" (distinct from the `exit 2` hard-block codes).

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
- **F9** dissolves because completed.json is permanent and never archived; its presence is
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
- completed.json as a permanent terminal record eliminates all reader/rerun steady-state
  ambiguity and survives cleanup.
- Framed checksummed intent log with matching-destination-hash recovery makes crash recovery
  both self-describing and idempotent.
- Advisory flock on stable inode eliminates lock-split race and makes stale-owner reclamation
  automatic (kernel-guaranteed on process death).
- Txn record separate from flock provides durable maintenance intent that blocks ordinary writers
  regardless of PID liveness.
- Authorization gate at single pivot point makes the authorization window unambiguous.
- Linux fd-binding exec (execveat AT_EMPTY_PATH) eliminates exec TOCTOU on Linux.
- macOS/darwin-arm64 (primary operator platform) residual TOCTOU window is documented,
  mitigated with programmatic mtime guard and hard pre-flight "no concurrent build" checklist
  item, and requires explicit POLICY 22 human acknowledgment at ratification.
- Platform-branched durability barriers correctly apply F_FULLFSYNC on macOS (Apple-documented)
  vs. fsync+dir-fsync on Linux (POSIX-standard).

### Negative

- Significantly more implementation complexity than v1.2: staging generation dir, framed intent
  log, advisory flock, txn record, CURRENT.json + completed.json pointer, platform-branched
  durability, guard-branch separation, fd-binding exec.
- macOS/darwin-arm64 (primary operator platform) exec TOCTOU residual window requires human
  acknowledgment at POLICY 22 ratification; mitigated by programmatic mtime guard and hard
  pre-flight "no concurrent cargo build" checklist item (v1.4 M5).
- APFS directory-fsync durability is unverified: empirical darwin-arm64 durability test required.
- Fault injection test suite between every rename/fsync/intent-record-write step adds test scope.

### Neutral

- BC-1.18.011 Amendment 6 correction (Linux-vs-macOS dir-fsync) changes the spec but not the
  fundamental migration approach.
- The Option B selection (one-time interactive Bash approval) is unchanged.
- The three-way ARCH-INDEX parity check (§Decision 10) is unchanged.

---

## Alternatives Considered

See §Rationale for the full head-to-head evaluation of Options A, B, and C.

- **Option A (Hook-driven explicitly-armed one-shot native action):** Viable; not chosen due to
  operational complexity for a one-time migration. Recommended for reconsideration if recurring
  migration use cases emerge.
- **Option B (One-time interactive Bash approval at F4) — CHOSEN:** Least privilege; one-time
  human gate; zero standing permission surface after migration completes; direct D-449(a)
  evidence capture; chosen per §Decision 1.
- **Option C (Hardened standing allowlist):** Fails the activation-authorization requirement
  (standing permission ≠ activation authorization per §Decision 4); post-migration cleanup
  burden; no governance benefit over Option B.

For the atomic-pointer model vs. incremental per-file rename: §Rationale
"Why v1.3 atomic-pointer model was adopted" explains the shared root cause across F1/F2/F5/F9
that made incremental amendment non-viable (diverging finding trajectory 7→8→11).

## Source / Origin

- BC-1.18.011 `S-25.02` cluster-5 specification (governs B2 body-split migration)
- BC-1.18.010 §Reader Integration (governs BC-INDEX reader protocol during migration window)
- S-25.06 Rule 7 (governed mechanism-A backfill-split activation; corrected by §Decision 9)
- In-house local adversary cascade — all passes recorded in decision log:
  - External pass-1 (`adv-cv-adr052-cluster5-F1-2026-09-12.md`, D-1214): 1st RATIFY-WITH-CHANGES
    (7 findings), producing v1.1 fix-burst
  - External pass-2 (`adv-cv-adr052-v11-closure-2026-09-12.md`, D-1216): 2nd closure review
    (8 findings), producing v1.2 redesign
  - External pass-3 (`adv-cv-adr052-v12-closure-2026-09-13.md`, D-1218): 3rd Codex cross-vendor
    (11 findings NOT RATIFIABLE), driving v1.3 full redesign
  - Local pass-1 (D-1221): 4th review (9 ADR-owned findings, 2C+5H+2M), producing v1.4 fix-burst
  - Local pass-2 (D-1222): 5th review (9 ADR-owned findings, 1C+4H+4M), producing v1.5 fix-burst
  - Local pass-3 (`adv-local-adr052-pass3.md`, D-1223): 6th review (12 findings NOT RATIFIABLE,
    1C+4H+5M+2L), producing this v1.6 fix-burst
- `research-adr-052-v13-atomic-publication-2026-09-13.md` — research brief grounding
  the atomic-pointer architecture, advisory flock design, and F_FULLFSYNC / fexecve guidance
- CLAUDE.md TD-FACTORY-HOOK-BYPASS-001 P0 — governing rule this ADR excepts via §Decision 8
- ADR-051 §Decision 1 (WASM fuel-budget constraint), §Decision 7 (B2 end-state);
  canonical sub-shard naming `BC-INDEX-SS-NN.a.md` / `.b.md` per ADR-051 §Decision 7

---

## Downstream to Product-Owner

The architect specifies the following amendments; the product-owner writes all BC body changes.
Do NOT modify BC content directly; route to product-owner.

### BC-1.18.011 required amendments (v1.3 updates — ordered by precedence)

**Amendment 1 — Precondition 4 (scheduling coupling removal — v1.5 H-1: inlined from v1.2):**

Replace the current Precondition 4 text with:

> "The migration is independently gated on the F4 activation boundary. It has NO timing or
> ordering dependency on BC-1.18.008's mechanism-A backfill-split; the two migrations activate
> independently (each via its own armed-activation manifest per ADR-052 §Decision 4) and may
> run in any order. They share an F4 activation window by operational convenience, not by
> specification."

**Amendment 2 — Postcondition 6 (A/B2 coupling removal — v1.5 H-1: inlined from v1.2):**

In the final sentence of Postcondition 6, replace "at the SAME F4 activation moment mechanism
A's own backfill (BC-1.18.008) runs" with "at F4 activation, as part of the same one-time B2
migration operation (independently of mechanism A's activation schedule)." The requirement that
SS-05/SS-06 sub-split occurs WITHIN the same B2 operation (not a separate follow-on) is
PRESERVED; only the A/B2 simultaneous-activation coupling is removed.

**Amendment 3 — Postcondition 7 scope clarification (v1.5 H-1: inlined from v1.2):**

Append to the end of Postcondition 7:

> "Note: this postcondition governs B2/Cohort-B independence only. A/B2 scheduling independence
> (that mechanism A and B2 activate independently at F4) is governed by Precondition 4 [as
> amended per ADR-052 §Decision 1]."

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
>    - State COMPLETED: all canonical path moves complete and hash-verified; completed.json
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

**Amendment 7 — Postcondition 3a: TOCTOU fingerprint recheck (v1.3 update; pointer-swap step
corrected to step 6 per v1.4 M2):**

Replace the Amendment 7 text from v1.2 with:

> "3a. **Pre-commit source-fingerprint recheck (TOCTOU guard).** This check is performed EXACTLY
>     ONCE, immediately before the CURRENT.json pointer swap (step 6 in ADR-052 §Decision 7c)
>     — not before the first rename. The migration binary re-reads BC-INDEX.md's source content,
>     computes SHA-256, and compares against the `source_sha256` field recorded in the txn record
>     at quiescence. If they differ: ABORT. The txn record is set to state ABORTED. No canonical
>     paths have been changed at this point (the pointer swap has not occurred). The migration
>     requires re-activation. This single-check design eliminates the v1.1 contradiction where a
>     re-check after BC-INDEX.md's own rename would find a fingerprint mismatch and incorrectly
>     trigger abort."

**Amendment 8 — Invariant 3 commit-pointer (v1.3 update; generation-first reader protocol added
per v1.5 C-1):**

Replace the Amendment 8 text from v1.2 with:

> "3. **The migration is never partially applied.** At every observable point in time — before
>    the migration runs, during staging, and after it completes — `BC-INDEX.md`'s body is either
>    the FULL original monolithic form or the FULL split end-state form; it is never observed in
>    a state where some subsystems are split and others are not. The all-or-nothing guarantee is
>    implemented via the CURRENT.json atomic pointer swap and the intent log: before the CURRENT.json
>    pointer swap (ADR-052 §Decision 7c step 6), the state machine is either STAGING (staging
>    generation in progress, intent log recording per-target expected hashes) or has no txn record;
>    after the CURRENT.json pointer swap transitions to COMMITTING, canonical readers see the
>    migration as in-progress and access new-generation content via the generation-first /
>    canonical-fallback protocol (ADR-052 §Decision 7c): try `gen-<uuid>/` path FIRST (file present
>    = not yet moved to canonical = new content); fall back to canonical path if absent from gen dir
>    (file already moved by step 7 progress = canonical now holds new content). This protocol is
>    correct for BOTH net-new shard files AND in-place-overwrite targets such as BC-INDEX.md.
>    ENOENT is impossible for any new-generation file during the COMMITTING window (rename(2)
>    atomicity). The CURRENT.json atomic pointer swap is the sole commit-point for the multi-file
>    atomic operation; composing N independent write_atomic calls without this commit-pointer does
>    not satisfy this invariant. completed.json (written at §Decision 7c step 8) is the permanent
>    terminal record; forward recovery uses the intent log + matching-destination-hash rule
>    (ADR-052 §Decision 7b) to resume from the first uncompleted canonical path move."

**Amendment 9 — Architecture Anchors (v1.3 update):**

Replace the Amendment 9 text from v1.2 with the following complete Architecture Anchors section:

> - `crates/factory-dispatcher/src/shard_manager.rs` — one-time B2 migration entry point reusing
>   BC-1.18.006's staging/atomic-replace primitives
> - `.factory/specs/behavioral-contracts/BC-INDEX.md` §Summary / `total_bcs` frontmatter field —
>   the independent count-oracle this BC's census check (Postcondition 2) cross-checks against
> - `.factory/specs/architecture/ARCH-INDEX.md` §Subsystem Registry — the BC-S Prefix→SS-NN
>   mapping this BC's per-subsystem partition boundaries follow
> - ADR-052 §Decision 4 — armed-activation manifest (two-phase validation: pre-lock and
>   under-exclusion)
> - ADR-052 §Decision 5a — native admission gate in executor.rs: OPEN/DRAINING gate with writer
>   reservations (PreToolUse-acquire/PostToolUse-release); txn record state check (STAGING/COMMITTING)
>   blocks ordinary writers regardless of PID liveness
> - ADR-052 §Decision 7a — advisory flock on stable pre-created never-unlinked inode
>   (`.factory/migration-state/exclusive.lock`); durable txn record separate from lock file with
>   `fencing_generation` (AUDIT-ONLY) for recovery-owner claim
> - ADR-052 §Decision 7b — framed checksummed intent log with per-target expected post-hash +
>   pre-state; WAL boundary after intent fsync; matching-destination-hash recovery decision table
> - ADR-052 §Decision 7c — single atomic CURRENT.json pointer swap (the commit point at step 6);
>   completed.json as permanent terminal record; generation-first/canonical-fallback reader protocol
> - ADR-052 §Decision 8 — POLICY 22 exception declaration with accurate skipped-control inventory
>   and enumerated allowed write targets

### BC-1.18.010 required amendments (v1.3 updates)

**Invariant 2 amendment:** UNCHANGED from v1.2. Apply verbatim.

**Reader integration amendment (v1.3 update — replaces v1.2 text):**

Replace the §Reader Integration section with the following (v1.5 M-4: removes stale v1.3 "use gen-uuid/" and stale v1.4 "canonical-first" text; single correct instruction per §Decision 7c C-1 fix):

> "During the B2 migration window (after CURRENT.json pointer swap, before completed.json
> written), readers accessing BC-INDEX paths MUST use the following protocol:
> 1. Check `.factory/migration-state/completed.json` — if exists: canonical paths are current.
> 2. Check `.factory/migration-state/CURRENT.json` — if `status: committing`: for each
>    required file, try `gen-<generation_id>/` path FIRST; if absent (already renamed to
>    canonical), fall back to canonical path. (generation-first / canonical-fallback per
>    ADR-052 §Decision 7c)
> 3. If neither CURRENT.json nor completed.json exists: legacy BC-INDEX.md path is current.
> In steady state (completed.json present), canonical paths are always authoritative."

### error-taxonomy.md correction (v1.3 update — replaces v1.2 text)

Replace the v1.2 correction with:

> "The maintenance lock (E-MAINTENANCE-001) is enforced by two independent mechanisms:
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
| Architecture Anchors (§Amendment 9) | References §Decision 7a PID+activation_id lock file; §Decision 7 per-file rename sequence | Replace with: §Decision 7a advisory flock + txn record; §Decision 7b intent log; §Decision 7c CURRENT.json pointer swap + completed.json |

**v1.4 additional changes to BC-1.18.011 (product-owner must apply):**

| Section | v1.3 text (to replace) | v1.4 change required |
|---|---|---|
| Invariant 3 | "CURRENT.json atomic pointer swap is the sole commit-point" — does not reflect C1 reader protocol change | Update: add that during COMMITTING state, new-generation content is accessible via canonical-first / generation-fallback protocol; a file is at canonical path (if already moved) OR at gen-uuid/ path (not yet moved); ENOENT is not possible for any file during the committing window (C1 fix) |
| Postcondition 1/2 (census gate cross-ref) | No explicit pre-pivot gate step referenced | Add cross-reference to ADR-052 §Decision 7c step 3b (pre-pivot content-preservation + census gate); state that PC1/PC2 are verified at step 3b against the staged generation BEFORE the pointer swap; failure at step 3b aborts cleanly per step 3c |
| Postcondition 3a (Amendment 7) / PC3a | "step 5 in ADR-052 §Decision 7c" cited as the pointer swap location | CORRECTION: the CURRENT.json pointer swap is at **step 6** (not step 5). Step 5 = fingerprint recheck; step 6 = CURRENT.json pointer swap (the commit point). Update all PC3a and related references from "step 5" to "step 6" |
| State names (any remaining PREPARED occurrences) | Any "PREPARED" in BC body or preconditions | Replace with STAGING. ADR-052 §7a enum authoritative: STAGING, COMMITTING, COMPLETED, ABORTED — no PREPARED state exists |
| Resume logic EC-003 | EC-003 resumes from first uncompleted intent log move | Add: EC-003 for resume-from-STAGING MUST RE-RUN the full census (step 3b) before proceeding to the pointer swap; resume is not permitted to skip the census gate |

**Note on M3 (stale total_bcs stdout re-grounding — BC-owned finding, not ADR-owned):**
BC-1.18.011 may have `total_bcs` stdout references that need updating to match current
BC-INDEX frontmatter values. This is a BC-owned finding for the product-owner to assess and
correct independently of the ADR changes.

### v1.5 additional BC changes (product-owner must apply)

**C-1 (generation-first reader protocol):**
- BC-1.18.010 §Reader Integration step 2: Replace with a single generation-first instruction per
  §Decision 7c: "for each required file, try `gen-<generation_id>/` path FIRST; if absent, fall
  back to canonical path." This supersedes all prior versions of this instruction.
- BC-1.18.011 Invariant 3: The "new-generation content is accessible via canonical-first /
  generation-fallback protocol" language (added in v1.4) MUST be corrected to "generation-first /
  canonical-fallback" — gen-uuid/ is tried first; canonical is the fallback for already-moved files.

**H-4 (version-pin cleanup — stable §Decision N form):**
- BC-1.18.010 body: Replace any "ADR-052 v1.N §..." version-pinned references with stable
  "ADR-052 §Decision N" form (e.g., "per ADR-052 §Decision 7c" rather than "per ADR-052 v1.3
  §Decision 7c").
- BC-1.18.011 body: Same — replace all "ADR-052 v1.N §..." pins with "ADR-052 §Decision N".
- error-taxonomy.md: Replace any "ADR-052 v1.3+" or "ADR-052 v1.4" version pins in
  E-MAINTENANCE-001 catalog entry with stable "ADR-052 §Decision 5a" or "ADR-052 §Decision 7c"
  anchors as appropriate.

**M-1 (ADR-052 traceability row in BC-1.18.011 Architecture Anchors):**
BC-1.18.011 Architecture Anchors section currently cites only ADR-051. It MUST add ADR-052 as
an additional anchor (alongside ADR-051), since ADR-052 §Decision 4–11 are the authoritative
source for the one-time B2 migration mechanics, crash-atomicity guarantees, reader protocol,
and executive write-exclusion model that this BC governs.

**L-1 (error-taxonomy header):**
The error-taxonomy.md header may not acknowledge the MIG and MAINTENANCE categories added by
v1.19–v1.21. Product-owner to audit the header summary against the current category list in
the file body and update accordingly, then cross-check category enumeration against prd.md
§5.1 if prd.md enumerates error categories.

**L-5 (BC-1.18.011 EC-003 step citation):**
BC-1.18.011 EC-003 (resume-from-STAGING error code) currently cites "§4e" as the authority for
the re-run-census requirement. The correct citation is ADR-052 §Decision 7c step 3b (the
pre-pivot content-preservation + census gate), not §Decision 4e (the manifest consumption /
recovery modes section which documents WHEN resume occurs, not the census gate mechanics).
Product-owner to update EC-003's §4e citation to "ADR-052 §Decision 7c step 3b".

### BC-1.18.010 — what must change to match v1.3

| Section | v1.2 text (to replace) | v1.3 change required |
|---|---|---|
| §Reader Integration | "COMMITTED absent → read legacy BC-INDEX.md"; COMMITTED archived at CLEANED | Replace entire section with v1.3 reader protocol (completed.json check first; CURRENT.json generation pinning second; legacy only if neither exists) |
| §Reader Integration steady state | "after CLEANED, shard paths are canonical; COMMITTED archived" | Replace: completed.json is permanent + authoritative; no archiving of any terminal record |
| Invariant 3 (if present) | References COMMITTED marker as commit-pointer | Update to: CURRENT.json pointer swap is the commit-point; completed.json is the terminal record |

**v1.4 additional changes to BC-1.18.010 (product-owner must apply):**

| Section | v1.3 text (to replace) | v1.4 change required |
|---|---|---|
| §Reader Integration step 2 | "if `status: committing`: use gen-uuid/ paths for reads" (v1.3 stale text); "try canonical path first; fall back to gen-uuid/" (v1.4 canonical-first — regression for BC-INDEX.md overwrite target) | v1.5 M-4: replace BOTH stale instructions with a single generation-first/canonical-fallback instruction per §Decision 7c C-1 fix: "try `gen-<generation_id>/` path FIRST; if absent, fall back to canonical path." This is correct for net-new shard files AND BC-INDEX.md (in-place overwrite target). |

### error-taxonomy.md — what must change to match v1.3

| Location | v1.2 text (to replace) | v1.3 change required |
|---|---|---|
| E-MAINTENANCE-001 definition (near original line 84) | "enforced by `validate-factory-path-staging` on Edit/Write"; "when exclusive.lock exists with alive PID" | Replace with: dual-mechanism: (1) txn record state STAGING/COMMITTING blocks all mutation tools via native gate regardless of PID; (2) DRAINING gate prevents new reservations; guard is secondary classifier only |

**v1.4 additional changes to error-taxonomy.md (product-owner must apply):**

Add catalog rows for all new error codes defined in §Error Code Semantics above. Each row
must include: code name, trigger description, severity, exit behavior. Codes to add:
`E-BINARY-INTEGRITY-FAILURE`, `RECOVERY_REQUIRES_REAUTHORIZATION`, `EXPIRY_ABORT`,
`FINGERPRINT_MISMATCH_ABORT`, `DRAIN_TIMEOUT_ABORT`, `ARCH_INDEX_PARITY_ABORT`,
`COMPLETION_MANIFEST_REJECTION`, `CENSUS_MISMATCH_ABORT`, `CONTENT_PRESERVATION_ABORT`,
`ALREADY_MIGRATED` (exit-0 non-error sentinel), `E-MAINTENANCE-001` (update existing entry
per v1.3 correction above).

---

## References

- `adv-local-adr052-pass3.md` — local cascade pass-3 (D-1223, this burst): 12 findings NOT RATIFIABLE (1C+4H+5M+2L), producing this v1.6 fix-burst
- `adv-cv-adr052-v12-closure-2026-09-13.md` — 3rd Codex cross-vendor closure review (11 findings, D-1218) that prompted the v1.3 redesign
- `research-adr-052-v13-atomic-publication-2026-09-13.md` — research brief grounding the v1.3 architecture (atomic publication, intent log, advisory flock, F_FULLFSYNC, fexecve/execveat)
- `adv-cv-adr052-v11-closure-2026-09-12.md` — 2nd Codex closure review, 8 findings (D-1216)
- `adv-cv-adr052-cluster5-F1-2026-09-12.md` — 1st Codex RATIFY-WITH-CHANGES verdict (7 findings, D-1214)
- Decision log D-1221 (local pass-1, 9 findings, producing v1.4), D-1222 (local pass-2, 9 findings, producing v1.5), D-1223 (local pass-3, 12 findings, producing v1.6)
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
| `crates/factory-dispatcher/src/executor.rs` | OPEN/DRAINING gate with writer reservations (PreToolUse-acquire/PostToolUse-release); durable reservation dir `.factory/migration-state/reservations/` (H2 v1.4); atomic admission under gate_state flock; every abort path flips gate → OPEN (H1 v1.4); crash-recovery gate reconciliation; stale reservation cleanup; txn record state check blocking mutations | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/src/shard_manager.rs` | Full v1.6 migration implementation: advisory flock; txn record; intent log (framed+checksummed); step 3b pre-pivot census gate — staging-integrity + PC1 (byte-for-byte reconstruct vs source_sha256) + PC2 (per-ID exactly-one-shard set check; EC-001 dup+drop detection); CURRENT.json pointer swap + generation-first/canonical-fallback reader; completed.json; 4-branch recovery; platform-branched durability (F_FULLFSYNC on macOS); fd-binding exec on Linux with /proc dependency note; programmatic mtime guard on macOS (M-2: mtime re-stat immediately before exec); stale-PID GC at start of every drain (M-5); all abort paths flip gate → OPEN; fencing_generation AUDIT-ONLY | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/tests/` | v1.6 test suite: advisory flock; txn record state machine; intent log write+recovery+fault-injection; step 3b census gate (staging-integrity + PC1 concat-SHA vs source_sha256 + PC2 ID-set + EC-001 dup+drop + E-SHD-005); CURRENT.json pointer swap + generation-first reader; completed.json permanence; all abort paths → gate OPEN; Branch 2 gate reconciliation (H-2 fault injection: crash between step 8 and gate→OPEN, then ALREADY_MIGRATED reconciles gate); reservation dir cross-process quiescence (H-2); stale-PID GC on first-run drain (M-5); 4-branch guard logic; conservative Bash admission (txn=STAGING or COMMITTING); digest mismatch abort; Linux execveat AT_EMPTY_PATH; macOS mtime guard (pre-exec re-stat) + freeze-build; platform durability; validate_write_target() dot-lowercase + rejection of hyphen-uppercase (C2 ratification test) | implementer (cluster-5 TDD) |
| `crates/factory-dispatcher/tests/darwin_arm64_durability_test.rs` | Empirical darwin-arm64 directory-fsync durability characterization test (required to validate APFS dir-fsync behavior; flags whether dir-fsync provides any power-loss protection beyond F_FULLFSYNC alone) | implementer (cluster-5 TDD) — darwin-arm64 CI required |
| `.factory/activation/factory-dispatcher.sha256` | SHA-256 of built binary | devops-engineer (cluster-5 activation) |
| `.factory/activation/` | Created at F4 by state-manager | state-manager |
| `.factory/migration-state/` | Created by migration binary at runtime; `exclusive.lock` pre-seeded by devops-engineer; `reservations/` subdir created by executor.rs for cross-process writer reservations (H2 v1.4) | migration binary (runtime) |
| `.factory/migration-audit/` | Created by state-manager post-migration | state-manager |

## Changelog

| Version | Date | Author | Change |
|---|---|---|---|
| 1.6 | 2026-09-13 | architect | Fix-burst resolving all 12 findings from 6th adversarial review (local cascade pass-3, D-1223): 1 CRITICAL + 4 HIGH + 5 MEDIUM + 2 LOW. C-1 — Step 3b census gate was a tautology: renamed sha256(staged)==expected_post_hash check as staging-integrity (what it actually is, not PC1); added true PC1 (reconstruct concatenation → compare SHA-256 against source_sha256 from txn record); rewrote PC2 as per-ID set check (each ID in EXACTLY ONE shard, ZERO in retained body; count comparison alone insufficient — EC-001 dup+drop case must abort). H-1 — §4e table re-keyed on TXN-RECORD state (STAGING/COMMITTING/COMPLETED/ABORTED) per BC-1.18.011 Invariant 3's discriminator; impossible "CURRENT.json status:staging" row deleted. H-2 — Branch 2 ALREADY_MIGRATED path now reconciles stale gate before exit 0: acquire gate LOCK_EX; if gate≠OPEN and completed.json present with no active txn: flip→OPEN; fault-injection test mandate added. H-3 — Source/Origin and References: removed all citations to non-existent adv-cv-adr052-v13-closure-2026-09-13.md; replaced with real provenance (local cascade D-1221/D-1222/D-1223; adv-local-adr052-pass3.md being written this burst); Status finding-count corrected. H-4 — §Downstream Amendments 7/8/9 placeholder text replaced with exact replacement text mirroring BC-1.18.011 v1.5 (CURRENT.json pointer swap as commit-point; intent log recovery; completed.json permanent terminal record). M-1 — §5a PreToolUse: explicit text that admission checks BOTH gate_state AND txn-record state; gate_state is the durable proxy; either gate≠OPEN OR active txn (STAGING/COMMITTING) → E-MAINTENANCE-001. M-2 — macOS verify_and_exec_binary code sample: mtime re-stat added immediately before exec_by_pathname call (prose-code gap from H-2 v1.4 closure). M-3 — step 3c and §Error Code Semantics aligned: CONTENT_PRESERVATION_ABORT for PC1 (concat-vs-source_sha256) failures; CENSUS_MISMATCH_ABORT for PC2 (ID-set) and E-SHD-005 (boundary) failures; binary surfaces process exit codes, not HookResult. M-4 — CLAUDE.md amendment: .factory/cycles/*/ wildcard replaced with the four exact append-log paths (decision-log.md, burst-log.md, lessons.md, session-checkpoints.md in v1.0-brownfield-backfill). M-5 — Stale-reservation GC moved to start of EVERY drain (step 1 of drain procedure), not only recovery startup. L-1 — H1 title: "(v1.4 Fix-Burst)" version pin stripped. L-2 — Casing: all path-bearing references aligned to lowercase completed.json. inputs[] + adv-local-adr052-pass3.md. |
| 1.5 | 2026-09-13 | architect | Fix-burst resolving 9 ADR-owned findings from 5th adversarial review (1 CRITICAL + 4 HIGH + 7 MEDIUM). C-1 — Reader protocol regression fixed: v1.4's canonical-first/generation-fallback inverted to generation-first/canonical-fallback; generation-first is correct for BOTH net-new shards AND BC-INDEX.md (in-place overwrite target whose canonical path holds OLD monolithic body until step 7's rename); fault-injection reader test mandate added. H-1 — ADR made self-contained: §Decision 6 audit-trail decision inlined from v1.2 (completed.json substituted for COMMITTED phase marker per v1.3 redesign); §Decision 8 skipped-control inventory table inlined from v1.2; §Downstream Amendments 1–3 fully inlined from v1.2; no "see v1.2 §" dangling references remain. H-2 — mtime guard corrected: re-stat moved to IMMEDIATELY BEFORE execve call; previous "after exec returns" fired only on exec failure (execve never returns on success); "Human sign-off required" updated with corrected residual-risk semantics (sub-instruction window, not digest-check-to-exec window). H-3 — Reservation UUID corrected: PreToolUse creates `<tool_use_id>.reservation` using stable harness tool-invocation ID shared across Pre/Post hook pair; PostToolUse removes it by same tool_use_id without shared in-process state; test mandates added (Pre-creates/Post-removes; stale-PID cleanup). H-4 — Load-bearing version pins removed: CLAUDE.md amendment text updated from "ADR-052 v1.4 EXCEPTION" to "ADR-052 EXCEPTION"; all "see v1.2 §" refs inlined (H-1); BC-impact handoff directs PO to use stable §Decision N form throughout. M-2 — ADR-051 §Decision 10 amended: "at the SAME F4 activation moment mechanism A's backfill runs" coupling removed; B2 sub-split occurs within same one-time B2 operation independently of mechanism A schedule; ADR-051 bumped to v1.14. M-3 — E-MAINTENANCE corrected to E-MAINTENANCE-001 throughout (error-taxonomy.md SoT). M-4 — §Downstream BC-1.18.010 §Reader Integration: stale v1.3 "use gen-uuid/ paths" instruction deleted; single correct generation-first/canonical-fallback instruction kept consistent with C-1. M-6 — Bash admission and reservation: explicit text added to §5a that admitted Bash mutations with write effect create `<tool_use_id>.reservation`; §5c classifier determines write-effect; quiescence waits for all Bash reservations; test mandate added. Observations: L-2 validate_write_target() positive test cases added for BC-INDEX.md and BC-INDEX.shard-manifest.toml; L-3 mechanism-A wildcard bounded by config-driven exact paths documented; L-4 30s drain-timeout liveness note added. BC impact for product-owner: C-1 generation-first reader update (BC-1.18.010 §Reader Integration + BC-1.18.011 Invariant 3); H-4 version-pin cleanup (BC-1.18.010/011 + error-taxonomy); M-1 ADR-052 traceability row to BC-1.18.011 Architecture Anchors; L-1 error-taxonomy header audit; L-5 BC-1.18.011 EC-003 step citation correction. inputs: fix: removed non-existent adv-cv-adr052-v13-closure-2026-09-13.md. |
| 1.4 | 2026-09-13 | architect | Fix-burst resolving 9 ADR-owned findings from 4th adversarial review (2 CRITICAL + 5 HIGH + 2 MEDIUM). C1 — Reader no-content window: reader protocol changed to canonical-first / generation-dir-fallback during committing; gen-uuid/ clarified as CONTENT-IMMUTABLE (not moved-out-immutable); readers never see ENOENT during step 7 progress. C2 — F10 allowlist normalization: dot-lowercase BC-INDEX-SS-NN.a.md / .b.md / .c.md; generic BC-INDEX-SS-NN.manifest.toml; validate_write_target() accepts dot-letter pattern; CLAUDE.md amendment regenerated; ratification-time test mandate added. H1 — Abort gate-stuck: every abort path now atomically flips gate to OPEN; crash-recovery reconciles stale LOCKED/DRAINING with no active txn to OPEN; fault-injection test mandate added. H2 — Cross-process writer reservation: per-event dispatcher model stated; active_writer_count replaced by durable reservation dir .factory/migration-state/reservations/; PreToolUse admission atomic under gate_state lock; coordinator polls dir for emptiness. H3 — Missing pre-pivot census gate: new step 3b (content-preservation + census gate) inserted between intent-log WAL and authorization gate; verifies PC1+PC2+E-SHD-005; on failure: ABORT, txn ABORTED, gate OPEN; resume-from-STAGING RE-RUNs census (EC-003). H4 — PREPARED state name: all PREPARED occurrences in §5c and F5/F7 rows replaced with STAGING; §7a enum STAGING\|COMMITTING\|COMPLETED\|ABORTED is authoritative. M1 — Fencing token audit-only: fencing_generation downgraded from enforcement to AUDIT-ONLY metadata; flock provides actual mutual exclusion; prove-authority language removed. M2 — Pivot step contradiction: §4d corrected to reference step 6 (not step 4) for pointer swap; auth gate=step 4, fingerprint=step 5, pointer swap=step 6; BC-1.18.011 PC3a step correction noted for PO. M5 — macOS primary operator platform: darwin-arm64 identified as primary runtime with residual TOCTOU; Linux is where TOCTOU is eliminated; no-concurrent-build elevated to hard checklist with programmatic mtime guard; /proc dependency documented. Error Code Semantics section added. BC Impact handoff expanded with v1.4-specific changes. |
| 1.3 | 2026-09-13 | architect | Full redesign per D-1218 (3rd Codex cross-vendor closure review, 11 findings, NOT RATIFIABLE). Adopts atomic-pointer architecture from research-adr-052-v13-atomic-publication-2026-09-13.md. Closes all 11 findings: F1/F9 — single atomic CURRENT.json pointer swap over immutable staging generation; completed.json as permanent terminal record; eliminates ambiguous-absence heuristic. F2 — framed checksummed intent log with per-target expected post-hash + pre-state; matching-destination-hash recovery; fail-closed recovery decision table; fault-injection test mandate. F3 — real OPEN/DRAINING admission gate with PreToolUse-acquire/PostToolUse-release writer reservations; wait for active_writer_count=0 before snapshot; no non-atomic shared→exclusive flock upgrade. F4 — advisory flock on stable pre-created never-unlinked inode; fail-closed on empty/corrupt metadata by acquire-first; automatic stale reclamation via kernel on process death. F5 — durable txn record separate from flock; ordinary writers blocked by txn state (STAGING/COMMITTING) regardless of PID liveness; recovery-owner bumps fencing_generation to claim ownership; releasing flock never deletes txn record. F6 — authorization gate moved to immediately before CURRENT.json pointer swap (single pivot); post-pivot COMMITTING state retained regardless of expiry; completion-only recovery manifest bound to old activation_id + staged-generation hash + fencing generation (replaces fresh activation). F7 — conservative Bash admission: block unknown-write-effect commands while txn record in STAGING/COMMITTING; classify sanctioned commands from config-driven target sets; canonicalization + alias rejection. F8 — four guard branches: census (no manifest), terminal-state no-op (completed.json check), new activation (full validation), recovery (completion-only manifest); manifest/flock required only for new activation and recovery branches. F10 — single authoritative allowed-write-targets list; CLAUDE.md amendment generated from that exact list with containment check at every write; sub-shards (.a.md/.b.md), BC-INDEX.shard-manifest.toml, BC-INDEX-SS-05.manifest.toml added. F11 — Linux: fexecve/execveat(AT_EMPTY_PATH) for fd-binding exec; macOS: fexecve UNAVAILABLE per Apple docs; decision: freeze build under maintenance lock + documented residual TOCTOU; human sign-off required at ratification. Platform durability: Linux fsync+dir-fsync (mandatory); macOS F_FULLFSYNC on file (mandatory per Apple docs); APFS directory-fsync best-effort only (Apple docs inconclusive); darwin-arm64 empirical durability test required. Corrects v1.2 Amendment 6 which incorrectly treated dir-fsync as mandatory on all platforms. |
| 1.2 | 2026-09-13 | architect | Full redesign per D-1216 (2nd Codex RATIFY-WITH-CHANGES 8 findings). Resolves: F1 — native admission gate added in `executor.rs` covering ALL mutation tools (Edit/Write/MultiEdit/Bash) before shard_cap_precheck, replacing the `^Bash$`-only guard-level check; drain protocol via TOCTOU abort. F2 — per-target completion tracking in PREPARED marker + single TOCTOU check before first rename (not repeated between renames) resolves COMMITTED-after-last-rename vs "original untouched" contradiction; reader integration protocol specified (COMMITTED marker as read-path selector for BC-1.18.010). F3 — content-bearing lock file with PID+activation_id separates persistent maintenance intent from OS advisory lock; pre-PREPARED crash recovery path defined; stale-lock recovery specified. F4 — two-phase manifest validation: pre-lock (lightweight) then under-exclusion (repo_root_sha, expected_total_bcs, three-way ARCH-INDEX parity, readiness); pre-publication expiry recheck before COMMITTED; manifest CONSUMED marking replaces deletion (resolves "absent=rejected" vs "already-migrated=exit0" contradiction); three expiry-safe recovery modes defined. F5 — explicit three-way activation-time parity check: config.arch_index_sha == manifest.approved_arch_index_sha == live ARCH-INDEX SHA; catches stale-binary case (revision A config + revision B manifest + revision B live). F6 — accurate skipped-control inventory: brownfield-discipline description corrected to ".reference/ write protection"; factory-branch-guard row added with explicit waiver; validate-factory-path-staged PostToolUse Bash corrected from "bypassed" to "MUST be amended." F7 — full-command pre-shell classifier moved to guard layer (validate-factory-path-staging + destructive-command-guard) with executable digest verification; removed impossible "binary rejects metacharacters post-shell-parse" claim; negative tests specified. F8 — CLAUDE.md amendment text expanded to include BC-INDEX.md + migration-state/ + activation/ + migration-audit/ + config-specified mech-A targets; preconditions (a-e) separated from post-success obligations (f-g); CLAUDE.md rule referenced by text anchor not line number; 4 guard amendments listed (vs 3 in v1.1). Downstream to Product-Owner: Amendment 5 corrected (Edit/Write → ALL mutation tools via native admission gate); error-taxonomy.md correction added; BC-1.18.010 §Reader Integration section added. NOT RATIFIED per D-1218. |
| 1.1 | 2026-09-12 | architect | Full revision per D-1214 (1st Codex RATIFY-WITH-CHANGES 7 findings + research Q1-Q5). Mechanism changed from Option C (hardened standing allowlist) to Option B (one-time interactive Bash approval at F4, no settings.json change). Option A re-evaluated honestly. Declares explicit narrowly-authorized policy exception (§Decision 8). Activation-manifest authorization mechanism added (§Decision 4). 9 PreToolUse `^Bash$` dispatcher guards enumerated (§Decision 5). Audit trail upgraded to factory-artifacts commit satisfying NIST AU-9 (§Decision 6). Crash-atomicity phase markers, TOCTOU pre-commit guard, dir-fsync mandate added (§Decision 7). Config snapshot bound to ARCH-INDEX revision with activation-time parity check (§Decision 10). BC-1.18.011 Amendments 1-9 and BC-1.18.010 Invariant 2 amendment specified. NOT RATIFIED per D-1216. |
| 1.0 | 2026-09-12 | architect | Initial authoring. Resolves CV-DIR-F2 (impossible execution path). Specifies Bash-tool-with-allowlist as sanctioned invocation path for both mechanism-A (S-25.06) and mechanism-B2 (BC-1.18.011) migrations. NOT RATIFIED per D-1214. |
