# PR #824 — Cycle-6 focused fresh-eyes confirmation review

- **PR:** #824 (S-25.02 cluster-2, BC-1.18.006)
- **Scope:** ONE delta only — commit `74bfbec8` (parent `54d6c84b`). The rest of the PR was APPROVED at `54d6c84b` (cycle-5).
- **Delta:** rewrite of `is_genuinely_missing` / `closest_existing_ancestor_is_directory_or_absent` in `crates/factory-dispatcher/src/shard_manager.rs` from a Windows-`raw_os_error()`-code-branching design to a portable ancestor-walk, plus two test fixes.
- **Reviewer model:** Opus 4.8 (cognitive diversity vs implementer)
- **CI note:** windows-x64 re-run on `74bfbec8` tracked by orchestrator; this review is code-correctness confirmation independent of CI.

## VERDICT: APPROVE

The delta is correct on both platforms, introduces no regression to the cycle-5-approved behavior, and its two revised tests are sound and load-bearing. No BLOCKING or SUGGESTION-grade finding. Two NITs (one doc-accuracy inconsistency introduced by the delta; one pre-existing documented deviation) are recorded below — neither gates merge.

---

## 1. Correctness of the ancestor-walk (both platforms)

**Verified correct.** Traced against every scenario class:

| Scenario | Unix | Windows | Result |
|----------|------|---------|--------|
| Traverse-through-a-file `/d/plainfile/child.md` | `read`→`NotADirectory` → early-return `false` (walk not reached) | `read`→`NotFound` → walk: `metadata(/d/plainfile)`→`Ok(is_dir=false)`→`false` | **propagate ✓** both |
| Missing leaf under existing dir `/d/new.md` | `NotFound` → walk: `metadata(/d)`→`Ok(dir)`→`true` | same | **relieve ✓** both |
| Multi-level missing `/d/a/b/c.md` (a,b absent) | walk strips `a/b`,`a` (NotFound), hits `/d`(dir)→`true` | same | **relieve ✓** both |
| Multi-level traverse-through-file `/d/plainfile/a/b.md` | `NotADirectory`→early `false` | walk: `metadata(.../a)`→NotFound strip, `metadata(/d/plainfile)`→`Ok(is_dir=false)`→`false` | **propagate ✓** — the walk's not-stop-at-first-missing-level property is what makes Windows correct here |

**Edge cases assessed:**
- **Relative bare filename** (`"foo.md"`): `parent()`→`Some("")`; the `candidate.as_os_str().is_empty()` guard returns `true` before probing `metadata("")`. Correct (CWD-relative → non-blocking).
- **Relative nested** (`"a/b.md"`): probes `metadata("a")` relative to CWD; falls through to empty-guard `true` if absent. Correct.
- **Root/prefix bottom-out** (`"/x.md"`, `"C:\\x.md"`): `metadata("/")`/`metadata("C:\\")`→`Ok(dir)`→`true`. Chain exhaustion (`parent()==None`) → loop exits → `true`. Correct.
- **Termination:** `candidate.parent()` strictly shrinks the path each iteration; terminates at `None` or the empty-component guard. No infinite-loop risk.

**On the raised PermissionDenied concern** (`Err(_) => strip level, continue` swallowing a non-NotFound ancestor error): assessed as **NOT a blocking defect**, for a chain of reasons:
1. `is_genuinely_missing` early-returns `false` unless the *original* operation's `err.kind() == NotFound`. A `PermissionDenied` on the original failing op therefore never routes into the walk at all.
2. For the *original* op to have returned `NotFound`, path resolution must already have traversed (permission-wise) up to the failing component. A non-NotFound (`PermissionDenied`) result from `metadata(ancestor)` on an ancestor at/below that already-traversed level is therefore only reachable via a **TOCTOU race** (the whole check is inherently TOCTOU-racy per research doc §5 note; it is idempotence classification, not a security boundary).
3. Even in that racy case the effect is `relieve → treat as first-write`. All 7 call sites are **reads** whose relief (`Ok(vec![])`/`Ok(0)`/`Ok(None)`/`Ok(false)`) is followed by write logic in `execute_roll`; a genuine permission problem re-surfaces at the subsequent write and maps to `ShardRollError::SealWriteFailed` (E-SHD-001). It is not silently swallowed permanently.
4. It is explicitly documented (NIT-3 doc block, lines 187–196).

See NIT-B below for the (non-gating) observation that this diverges from the research doc's own prescribed algorithm.

## 2. No regression to cycle-5-approved behavior

**Confirmed — no regression.**
- **Signature unchanged** (`is_genuinely_missing(&io::Error, &Path) -> bool`); all **7 production call sites** (lines 1188, 1383, 2483, 2587, 3550, 3676, 3716) call identically. Diff touches only the two functions + 4 tests — nothing in the MAJOR-1/2/3 or FIX-MED-1 TOCTOU-adjacency regions (`create_new_exclusive` etc.).
- **Unix behavior-equivalence:** cycle-5 Unix path was `#[cfg(not(windows))] { true }` (unconditional relief on any `NotFound`). The new code walks instead. For **every real Unix case** the walk yields the identical result: a genuine Unix `NotFound` only arises when all existing ancestors are directories (traverse-through-a-file is `NotADirectory`, filtered at the early return and never reaching the walk), so the walk resolves to `true` — same as before. The walk is strictly *safer* in racy edge cases (would propagate rather than blindly relieve). Cost: one extra `fs::metadata` syscall per Unix relief — negligible.
- **cfg simplification:** the helper's former `#[cfg(any(windows, test))]` gate is removed; it is now compiled and used on every platform. `path` is used unconditionally (no dead `let _ = path`). Build + clippy clean (no dead-code / unused-var warnings).

## 3. Test soundness

**Both revised tests are load-bearing and cross-platform-correct.**

- `test_FINDING1_..._path_traversal_through_non_directory` — dropping `assert_ne!(err.kind(), NotFound)` **loses no meaningful coverage**: that assertion tested std's *platform error taxonomy* (and was Unix-only — it is false on Windows for this identical fixture, which is exactly why cycle-5 failed windows-x64). The retained `assert!(!is_genuinely_missing(&err, &path))` tests *our function's behavior* and holds on both platforms (Unix: early-return via `NotADirectory`; Windows: walk hits the real plain-file ancestor). This is precisely research-doc §"Test-fixture portability" option 1 (assert semantic outcome). Load-bearing on both platforms.
- `test_BC_1_18_006_P1a_..._genuine_non_notfound_...` — cfg-gated fixture is correct: Unix keeps the traverse-through-file→`NotADirectory` fixture; Windows uses the illegal-`<`-char→`ERROR_INVALID_NAME`→`InvalidFilename` fixture (research-doc §"option 2", source-verified against `decode_error_kind`). Both assert `expect_err` **and** `assert_ne!(kind, NotFound)` — the sanity precondition remains genuinely non-`NotFound` on both, so the test still exercises the "other-kind failure propagates, not relieved" contract of `read_canonical_content`. Load-bearing.
- Windows-only synthetic tests (`from_raw_os_error(2)`/`(3)`) remain valid: they synthesize `NotFound`-kind errors and now exercise the walk via real path fixtures (empty parent → relieve; real plain-file ancestor → propagate; missing-parent-under-real-dir → relieve). Consistent with the new algorithm.

**Local test evidence (macOS/Unix):** `is_genuinely_missing` filter 3/3 ok; `closest_existing_ancestor` filter 3/3 ok; `P1a` filter 3/3 ok. `cargo build -p factory-dispatcher` ok; `cargo clippy -p factory-dispatcher --all-targets` clean. Windows-cfg tests reasoned through (not executable on this runner); CI re-run tracks the live Windows result.

## 4. New findings introduced by this delta

### NIT-A (doc accuracy — introduced by this delta)
The test-region banner comment (shard_manager.rs ~lines 4151–4155) still reads: *"the path-aware helper is unconditionally compiled under test (`#[cfg(any(windows, test))]`)"*. The delta **removed** that `#[cfg]` gate — the helper is now compiled unconditionally on *every* build, as the function's own updated doc comment (lines 178–179) correctly states. The banner now contradicts the function doc.
- **Severity:** NIT (comment-only; no behavioral impact).
- **Proposed routing:** `vsdd-factory:implementer` — update the banner to match (drop the `#[cfg(any(windows, test))]` citation). Non-gating; can ride a later sweep.

### NIT-B (research-alignment — pre-existing, NOT introduced by this delta)
The walk's `Err(_) => cur = candidate.parent()` arm diverges from the research doc's own §5 step-2 prescription (*"`Err(other)` (e.g. `PermissionDenied`) → propagate the error, do NOT claim missing"*). The implementation instead treats every ancestor-`metadata` error as "absent" and keeps walking. As analysed in §1 this is inert in practice and is documented as NIT-3; the arm predates commit `74bfbec8` (the delta only reworded the surrounding NIT-3 comment). Production-grade hardening would be a 2-line change (`Err(e) if e.kind()==io::ErrorKind::NotFound => cur = candidate.parent(), Err(_) => return false`) which is strictly more conservative and costs nothing on the common relief path.
- **Severity:** NIT / low-value hardening. Not gating this delta (pre-existing, documented, inert, non-silent downstream).
- **Proposed routing:** `vsdd-factory:implementer` if the orchestrator elects to bring the code into exact parity with the source-verified algorithm; otherwise the existing NIT-3 documentation is an acceptable standing record.

---

## Finding table

| # | Severity | Category | Finding | Suggestion | Routing |
|---|----------|----------|---------|-----------|---------|
| NIT-A | nit | coherence (doc) | Test-region banner still cites removed `#[cfg(any(windows, test))]` gate | Update banner to match unconditional compilation | implementer |
| NIT-B | nit | correctness (research parity) | Walk swallows non-NotFound ancestor errors vs research §5 step-2 (inert, documented as NIT-3, pre-existing) | Optional 2-line hardening to propagate non-NotFound | implementer (optional) |

No BLOCKING findings. No SUGGESTION-grade findings. Delta APPROVED.

## What was verified (anti-rubber-stamp)
- Diffed `54d6c84b..74bfbec8`; read final state of both functions + all 4 revised tests + `read_canonical_content` call site + the 7 production call sites.
- Traced the ancestor-walk across 4 scenario classes × 2 platforms + 3 edge-case classes (relative, root/prefix, empty-component) + termination proof.
- Independently assessed the PermissionDenied `Err(_)`-swallow concern against the early-`NotFound`-filter invariant and the downstream write-error re-surfacing; concluded inert/non-silent.
- Confirmed Unix behavior-equivalence to cycle-5 (unconditional-`true` → walk yields identical result for all real cases).
- Ran the affected tests (9/9 ok on Unix), built the crate, ran clippy (clean).
