---
document_type: cross-vendor-closure-review
adversary_model: openai-codex
type: cross-vendor-closure-review
streak_impact: NONE
reviewed_at: 2026-09-13
reviewed_version: ADR-052 v1.2
verdict: NOT-RATIFIABLE
ratify: false
note: "BC-5.39.001 cycle streak 3/3 UNCHANGED — cross-vendor closure reviews are NON-STREAK (decision-support only). This is the 3rd cross-vendor Codex closure review of ADR-052 v1.2. Direction decision escalated to human."
---

# 3rd Cross-Vendor Codex Closure Review: ADR-052 v1.2

**Date:** 2026-09-13
**Adversary model:** openai-codex (decision-support; NON-STREAK)
**Prior review:** `adv-cv-adr052-v11-closure-2026-09-12.md` (2nd Codex CV, 8 findings, D-1216)
**Reviewed package:** ADR-052 v1.2 + BC-1.18.011 v1.2 + BC-1.18.010 v1.4 + error-taxonomy.md v1.19

## Reviewed Files

- `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md`
- `.factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md`
- `.factory/specs/behavioral-contracts/ss-01/BC-1.18.010.md`
- `.factory/specs/prd-supplements/error-taxonomy.md`

## Verdict

**NOT RATIFIABLE — DO NOT ratify v1.2 as written.**

**Novelty: HIGH.** The v1.2 revision package addressed the 8-finding mandate from the 2nd Codex review (D-1216) but did not achieve ratification quality. Eleven new findings (10 HIGH + 1 MED) were identified, spanning atomic publication, interrupted-rename recovery, writer exclusion, lock ownership, expiry handling, and policy scope. The trajectory is DIVERGING: 7 findings (1st review, D-1214) → 8 findings (2nd review, D-1216) → 11 findings (3rd review, D-1218).

## Trajectory Note

Cross-vendor finding counts: **7** (1st review, D-1214) → **8** (2nd review, D-1216) → **11** (3rd review, D-1218) — **DIVERGING**. The v1.2 redesign introduced additional specification surface in the atomic publication protocol that exposed more gaps rather than converging toward ratification quality.

## Root-Cause Read

Findings 1, 2, 6, and 9 stem from the same in-place per-file-rename publication model: renaming files one by one through an ordered sequence cannot provide an atomic view boundary regardless of journaling, because a reader or writer observing any intermediate state sees a partially-committed generation. Codex recommendations across all 11 findings converge on three structural changes:

1. **Generation-based atomic-pointer publication** — write all output to a new immutable generation directory, then atomically update a single pointer (symlink or file) to the new generation.
2. **OS advisory lock on a stable inode** — use a lock file that is never unlinked for ownership semantics; separate durable intent metadata from lock ownership.
3. **Immutable-executable digest binding** — bind the activation executable's digest into the approved activation record and prevent replacement between verification and execution.

Until these structural changes are adopted, incremental amendments to the per-file-rename model will continue to accrue findings.

## Direction Decision (Escalated to Human)

The DIVERGING trajectory and structural root-cause read above indicate that:
- **(a) v1.3 atomic-pointer redesign** (implement generation-based publication + OS advisory lock on stable inode + immutable-executable digest binding) is the path to ratification.
- **(b) Research-first** (investigate platform guarantees for the atomic-pointer approach before committing to full redesign) is a lower-risk alternative.
- **(c) Reconsider ADR-052 scope** (narrow the sanctioned-execution scope to avoid the publication problem entirely) is a third option.

This decision is escalated to the human. Cluster-5 TDD remains BLOCKED until POLICY 22 ratification of a clean Codex re-review.

## Findings (11 total: 10 HIGH + 1 MED)

### Finding 1 — Preserve an immutable legacy read generation until publication
**Severity:** HIGH | **Category:** consistency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:445`

**Evidence:** Lines 445–457 rename every target before writing COMMITTED: "COMMITTED is written AFTER the last rename". Lines 476–479 nevertheless require "COMMITTED absent: ... read from legacy BC-INDEX.md". BC-1.18.010:218–224 repeats this rule. Once BC-INDEX.md itself is replaced, a reader observing absent COMMITTED receives the new redirect body rather than the promised legacy rows. Making that rename last still leaves this window, including indefinitely after a crash.

**Recommendation:** Publish immutable generations through one durable atomic pointer, and require each reader to resolve and retain one generation for its entire operation. Alternatively retain a separately addressed immutable legacy snapshot until readers can safely transition. Specify reader behavior at every rename and marker boundary.

### Finding 2 — Recover renames that completed before their journal update
**Severity:** HIGH | **Category:** edge-case | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:446`

**Evidence:** Lines 446–451 order operations as rename, target-directory fsync, then append to completed_renames. Lines 452–454 claim targets not listed "are untouched"; lines 466–467 resume those targets using their staging paths. A crash after rename but before the journal update leaves a replaced target, an absent staging file, and no completed entry. A directory-fsync failure can leave the same state. The specified retry cannot recover it.

**Recommendation:** Persist each target's expected content hash and rename intent before publication. On restart reconcile both destination and staging content, accepting a matching destination as completed even without a completion entry, and fail closed on ambiguous states. Specify durable ordering for staging data and journal updates, and fault-inject between every rename, fsync, and journal write.

### Finding 3 — Drain admitted writers before reading migration inputs
**Severity:** HIGH | **Category:** concurrency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:264`

**Evidence:** Lines 264–270 call the fingerprint check a "Drain protocol" but only detect writes completed "between census and TOCTOU check". Lines 435–443 explicitly perform that check exactly once before the first rename. A writer admitted before lock creation can remain delayed until after this check and then mutate the source or a published target. Nothing in this protocol waits for that writer to finish, so its update can be lost or invalidate the verified output.

**Recommendation:** Introduce writer reservations held through actual mutation completion, stop new admissions, and wait for all admitted writers to drain before taking the source snapshot. Define how reservations span PreToolUse through tool completion and recover from tool failure. Keep the fingerprint check as defense in depth.

### Finding 4 — Serialize stale-lock reclamation and handle incomplete lock creation
**Severity:** HIGH | **Category:** concurrency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:379`

**Evidence:** Lines 379–382 require JSON ownership data in a file acquired with O_CREAT | O_EXCL. Lines 388–392 reclaim a stale lock by "Remove the lock file" followed by reacquisition. Two reclaimers can both inspect the old dead owner; one replaces the lock, then the other unlinks that new live owner's lock and also acquires it. Creation also exposes an empty file before its JSON is written, but §7a supplies no malformed/empty-lock disposition.

**Recommendation:** Use an OS advisory lock on a stable inode that is not unlinked for ownership, with separately published durable intent metadata. Define fail-closed handling for empty, corrupt, and unreadable metadata. If retaining a file-creation lock, specify a serialized reclamation protocol that cannot unlink another owner's replacement.

### Finding 5 — Define exclusive takeover of a dead owner's PREPARED transaction
**Severity:** HIGH | **Category:** consistency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:383`

**Evidence:** Lines 383–389 classify the lock as held when its activation_id matches PREPARED, even with a dead PID, and permit stale removal only when no PREPARED matches. There is no takeover step for the ordinary post-PREPARED crash. Meanwhile §5a:258–261 calls a dead-PID lock stale, and BC-1.18.011:85–86 plus error-taxonomy.md:84 block writers only for an alive PID. Recovery ownership and writer admission therefore disagree on this persistent transaction.

**Recommendation:** Separate process ownership from persistent maintenance intent. Keep ordinary writers blocked for every unresolved PREPARED transaction regardless of PID liveness, and specify an exclusive recovery-owner acquisition that updates ownership without clearing intent. Define abort-after-partial-publication behavior so releasing process ownership never exposes an unresolved transaction to writers.

### Finding 6 — Retain transaction recovery state when authorization expires
**Severity:** HIGH | **Category:** consistency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:210`

**Evidence:** Lines 212–215 check expiry immediately before PREPARED → COMMITTED and mandate deleting PREPARED and discarding staging on expiry. Lines 456–457 place that transition after all target renames, so expiry can delete the recovery record after canonical content has changed. Lines 227–231 additionally require a matching activation_id for recovery but demand a new activation manifest after expiry; line 167 defines that ID as unique per activation. No transition authorizes a new activation to recover the old transaction.

**Recommendation:** Check authorization before the first destructive publication step. After any rename, retain durable transaction state and maintenance intent until committed or safely rolled back. Define a fresh approval explicitly referencing the old transaction ID and immutable staged-output hashes, with clear rules for expiry during forward recovery.

### Finding 7 — Define how the native gate classifies Bash mutation targets
**Severity:** HIGH | **Category:** spec-gap | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:254`

**Evidence:** Lines 255–256 condition admission on an Edit/Write/MultiEdit/Bash operation whose "target path is under" either protected directory. However, lines 144–146 explicitly make migration targets config-driven with "no path arguments", and §5c receives a command string. The admission algorithm never defines how Bash commands yield target paths or how unknown effects are handled. Naming Bash in the tool list does not supply that classification.

**Recommendation:** Specify conservative Bash admission: while maintenance intent exists, block commands with unknown write effects and explicitly classify the sanctioned migration commands from their config-driven target sets. Define canonicalization and alias handling for direct tool paths, and ensure only an exclusively authorized recovery invocation can bypass the maintenance block.

### Finding 8 — Make census and completed reruns reachable through the guard stack
**Severity:** MED | **Category:** consistency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:322`

**Evidence:** Lines 322–326 require the classifier to read and validate an unexpired manifest after matching any permitted command. But lines 233–234 say --census requires no manifest, and lines 225–226 promise COMMITTED reruns exit 0 "regardless of manifest state". After manifest archival, both legitimate operations are rejected before the binary can implement those guarantees.

**Recommendation:** Specify separate guard branches for read-only census, validated terminal-state no-op, new activation, and recovery. Apply manifest requirements only to operations authorized to mutate, retaining exact-command and executable verification in every branch.

### Finding 9 — Retain an unambiguous terminal publication record after cleanup
**Severity:** HIGH | **Category:** completeness | **Confidence:** HIGH
**Location:** `.factory/specs/behavioral-contracts/ss-01/BC-1.18.010.md:218`

**Evidence:** Lines 221–224 say absent COMMITTED selects legacy BC-INDEX.md, but also that COMMITTED is archived at CLEANED and its absence then is normal. ADR §7b:408–424 puts PREPARED, COMMITTED, and CLEANED in one state file, while §4e:224–232 defines reruns only for COMMITTED, PREPARED, or no markers. The durable state determining that a fresh reader or rerun is now in steady state is not integrated into either decision algorithm.

**Recommendation:** Keep a permanent published-generation or terminal-success record at a stable location. Explicitly handle CLEANED in reader and CLI state tables, preserve all data needed for idempotent census output, and specify crash-safe ordering between cleanup, archival, and terminal-state publication.

### Finding 10 — Authorize all required migration output paths consistently
**Severity:** HIGH | **Category:** consistency | **Confidence:** HIGH
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:509`

**Evidence:** Lines 509–518 authorize ONLY BC-INDEX.md and shards/BC-INDEX-SS-NN.md for B2, plus the listed operational directories. BC-1.18.010:83 requires shards/BC-INDEX.shard-manifest.toml, :97 requires BC-INDEX-SS-05.manifest.toml, and BC-1.18.011:156 requires .a.md/.b.md sub-shards. These do not match the allowed shard filename. The proposed CLAUDE.md amendment at ADR:535–540 instead permits the entire shards directory, creating a different authorization scope.

**Recommendation:** Use one authoritative allowlist covering top-level and subsystem manifests, sub-shards, and staging/backup files required by the chosen publication protocol. Generate the ratification amendment from that same scope and require containment checks on every resolved target.

### Finding 11 — Bind executable verification to the actual executed artifact
**Severity:** HIGH | **Category:** security | **Confidence:** MEDIUM
**Location:** `.factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md:327`

**Evidence:** Lines 327–330 hash target/release/factory-dispatcher in a pre-shell guard and compare it to a separately stored build-time digest. Lines 106–111 subsequently execute that pathname through Bash with interactive approval. The specified admission boundary protects only behavioral-contract and cycle paths (:255–256), so it does not prevent replacing the release executable after verification. A concurrent rebuild or replacement can therefore make the approved command execute different bytes.

**Recommendation:** Publish the activation executable as an immutable artifact, bind its digest into the approved activation record, and prevent replacement until execution completes. Alternatively use a trusted launcher that verifies and executes the same opened artifact with a platform-specific mechanism. Explicitly document and test the verification-to-execution boundary.

## Decision Reference

D-1218 (STATE.md Decisions Log / decision-log.md): ADR-052 v1.2 NOT POLICY-22-ratifiable per this 3rd Codex closure verdict. 11 findings (10 HIGH + 1 MED), novelty HIGH. Trajectory DIVERGING (7→8→11). Direction decision escalated to human: v1.3 atomic-pointer redesign vs. research-first vs. reconsider ADR-052 scope. BC-5.39.001 cycle streak 3/3 UNCHANGED.
