# Security Review — PR #818

**Story:** S-25.02 cluster 1 — cap formula + native shard-cap trigger (BC-1.18.005)
**Branch:** `feature/S-25.02-cap-trigger`
**Reviewed at:** HEAD `d9eeb9bc5c651e0e9278d26e61c8c9d37b96d19f` (pre-review-convergence cycle; findings below remain accurate through the final merged head `f2769c245285a5582dbeecf7089b91c38e0a459d` — no security-relevant code paths were touched by later fix cycles beyond hardening the same mechanisms)
**Reviewer:** `vsdd-factory:security-reviewer` (Step 4 of pr-manager 9-step lifecycle)

## Scope

Full diff (`gh pr diff 818`) at time of review. New production code: `crates/factory-dispatcher/src/shard_manager.rs` (new file), `crates/factory-dispatcher/src/executor.rs` (+152 lines: `shard_cap_precheck`, `shard_gate_block_outcome`, wiring into `execute_tiers`), `crates/factory-dispatcher/src/lib.rs` (re-exports), `Cargo.toml`/`Cargo.lock` (new deps: `serde_norway` 0.9.42, `vsdd-hook-sdk`). Also reviewed: `crates/factory-dispatcher/tests/bc_1_18_005_shard_cap_trigger_test.rs`.

## Findings

### SEC-001: Unbounded file read before parsing untrusted-size frontmatter YAML
- **Severity:** LOW
- **CWE:** CWE-400 (Uncontrolled Resource Consumption)
- **Location:** `shard_manager.rs`, `read_changelog_item_count` — `std::fs::read_to_string(target_path)` with no size check beforehand.
- **Status: FIXED** during review convergence (cycle-2 finding B2/M-2). The read is now bounded via a single `File::open().take(MAX_CHANGELOG_TARGET_READ_BYTES + 1).read_to_string()`, closing both the unbounded-read and a TOCTOU gap between a separate `stat()` and the read. Verified by two independent fresh-context reviewers against the final code.
- Independently verified `serde_norway` 0.9.42 is not vulnerable to YAML "billion laughs" (inherits `serde_yaml`'s alias-jump budget and recursion-depth cap). No CVE/RustSec/GHSA advisory filed against it.

### SEC-002: Artifact matching by filename stem only, no path containment check
- **Severity:** LOW / INFO
- **CWE:** CWE-706 (Use of Incorrectly-Resolved Name or Reference)
- **Location:** `find_matching_entry` matched purely on `file_stem()` string equality, ignoring the directory component entirely.
- **Status: FIXED** during review convergence (cycle-2 finding B-1/M1, further hardened in cycle-3 finding S-2 for `./`-prefix normalization and cycle-4 finding F-1 for empty/CurDir-only path rejection). `ShardEntry` now carries a required `artifact_path` field; matching requires stem equality AND path containment via `path_falls_under_or_equals`, which correctly normalizes `Component::CurDir` and fails loud (`ShardConfigError::EmptyArtifactPath`) on a path that would otherwise vacuously match everything. Verified by five independent fresh-context review passes across the convergence loop.

### SEC-003: Diagnostic error messages include local file paths and config content
- **Severity:** INFO
- **CWE:** CWE-209 (Generation of Error Message Containing Sensitive Information)
- **Location:** `ShardConfigError` variants and stat()/read failure paths embed `artifact_stem`, absolute file paths, and raw parser error text into `HookResult::Error` messages.
- **Status: ACCEPTED, no action required.** No remote/cross-trust-boundary exposure — this is a local CLI/dev-tool hook consumed by the same operator who owns the filesystem being described, consistent with this project's existing dispatcher error-surfacing pattern elsewhere (e.g. `registry.rs`).

### SEC-004: Theoretical, non-exploitable integer-overflow surface in signed-delta cast
- **Severity:** INFO
- **CWE:** CWE-190 (Integer Overflow or Wraparound) — theoretical only
- **Location:** `net_delta_bytes_for_edit` — `new_len_bytes as i64 - old_len_bytes as i64`.
- **Status: ACCEPTED, no action required.** Not practically reachable (would require a single JSON payload field several orders of magnitude larger than any realistic process memory/IPC limit; Rust's `as` int-to-int cast is a well-defined truncation, not UB). Reviewer independently verified the rest of the cap-formula arithmetic (`compute_shard_cap_bytes`, `validate_entry`'s divisor-door closures, `projected_size_edit`'s saturating arithmetic, `validate_low_water_mark`'s i128-widened boundary comparison) is unusually well-defended against overflow/underflow/NaN/Infinity — this one inconsistent spot reads as an oversight, not a live risk.

### No findings in
- Command/shell injection (CWE-77/78) — no `std::process::Command` or shell-out anywhere in this diff.
- `unwrap()`/`expect()` panics on untrusted input — none in production code paths; all `#[allow(clippy::expect_used, ...)]` usage confined to `#[cfg(test)]` modules.
- Auth/permission bypass — N/A, no auth boundary introduced; runs inside the dispatcher's existing PreToolUse trust boundary. The gate currently only ever returns `Continue` or a fail-loud config-diagnostic `Error` (roll/block is explicitly deferred to BC-1.18.006/BC-1.18.009), so there is no live enforcement to bypass yet.
- Cryptographic misuse — N/A.
- Dependency vulnerabilities — both new dependencies checked clean (see SEC-001 note; `vsdd-hook-sdk` is an internal workspace crate, no external CVE surface).

## Summary

- **Total findings:** 4 (SEC-001..SEC-004).
- **Severity at time of review:** CRITICAL: 0, HIGH: 0, MEDIUM: 0, LOW: 2 (SEC-001, SEC-002), INFO: 2 (SEC-003, SEC-004).
- **Post-convergence status:** SEC-001 and SEC-002 (the only non-INFO findings) were both fixed during the subsequent review-convergence loop (5 total review cycles), independently re-verified against the actual code by multiple fresh-context reviewers each time, not merely asserted.
- **Blocking status at merge:** None. No CRITICAL/HIGH findings at any point; the two LOW findings were fixed and independently re-verified before merge.
