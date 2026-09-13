---
document_type: local-adversarial-review
adversary_model: claude (in-house fresh-context adversary agent)
type: local-adversarial-review
streak_impact: STREAK-BEARING (BC-5.39.001 LOCAL cluster-5 cascade)
reviewed_at: 2026-09-13
reviewed_version: ADR-052 v1.5
verdict: NOT-RATIFIABLE
pass: 3
sources:
  - .factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md
  - .factory/specs/behavioral-contracts/ss-01/BC-1.18.011.md
  - .factory/specs/behavioral-contracts/ss-01/BC-1.18.010.md
  - .factory/specs/prd-supplements/error-taxonomy.md
---

# In-House Local Adversarial Review — ADR-052 v1.5 package — Pass 3

Verdict: NOT-RATIFIABLE. Novelty: HIGH. Streak: LOCAL cluster-5 cascade pass-3 (BC-5.39.001).
Reviewed: ADR-052 v1.5, BC-1.18.011, BC-1.18.010, error-taxonomy v1.22; cross-checked ADR-051, ARCH-INDEX, BC-INDEX, VP-INDEX.

## CRITICAL

- C-1 — Step 3b "content-preservation (PC1)" is a self-referential tautology (expected_post_hash = sha256(staging_file), then verify sha256(staged)==expected_post_hash). BC-1.18.011 PC1 byte-for-byte reconstruction + PC2/PC2a exactly-one-shard census NOT enforced; EC-001 (dup-one/drop-one, same count) would pass the pivot. Governance-integrity gate inert. Fix: reconstruct+compare vs source_sha256; enumerate ID set exactly-one-shard.

## HIGH

- H-1 — §4e resume table keys on CURRENT.json status:staging, which is never written (CURRENT.json first written at step-6 pivot); STAGING crash matches no row. Fix: re-key on txn-record state.
- H-2 — Crash between completed.json write and gate→OPEN flip leaves gate permanently LOCKED; ALREADY_MIGRATED fast-path never acquires flock so never reconciles. Fix: terminal-no-op path must reconcile stale gate before exit 0.
- H-3 — ADR body cites non-existent adv-cv-adr052-v13-closure-2026-09-13.md; no source file for reviews (in-house passes not persisted). Fix: persist passes + cite real provenance.
- H-4 — §Downstream Amendments 7/8/9 defer exact text to "next burst" (placeholder). Fix: inline now.

## MEDIUM/LOW

- M: admission discriminator ambiguity (gate_state vs txn record); macOS code sample omits pre-execve mtime re-stat; census abort triple-coded (E-SHD-005 vs CENSUS_MISMATCH_ABORT vs CONTENT_PRESERVATION_ABORT); stale "(v1.4 Fix-Burst)" in H1 title; mechanism-A CLAUDE.md `*/` wildcard not human-auditable; stale-reservation GC only at recovery; finding-count mismatch; COMPLETED.json vs completed.json casing.
- [process-gap] BC→ADR "verified at step 3b" cross-ref never mechanically checked (same class as the E-SHD Display-drift lint gap, D-1221-PG-001 / S-12.13).

## Confirmed sound (not findings)

Reader protocol generation-first consistent across ADR+BCs; ADR-051 coupling removed; state-name enum consistent; 5-leg parity holds; SDK-grounding greps resolve; BC-INDEX titles match H1.

---

(All C-1/H-1..H-4 + MEDs were resolved in the ADR-052 v1.6 burst; this file is the provenance record.)
