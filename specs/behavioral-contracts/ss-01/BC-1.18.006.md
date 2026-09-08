---
document_type: behavioral-contract
level: L3
version: "1.5"
status: draft
producer: product-owner
timestamp: 2026-09-07T00:00:00Z
phase: F2
inputs:
  - .factory/specs/architecture/decisions/ADR-051-layer-2-two-mechanism-size-triggered-shard-rotation-append-logs-and-bc-index-sharding.md
  - .factory/specs/behavioral-contracts/ss-01/BC-1.18.005.md
  - crates/hook-sdk/src/result.rs
  - .factory/cycles/v1.0-brownfield-backfill/S-25.02-f2-architecture-delta.md
input-hash: "d39e2f4"
traces_to: .factory/specs/prd.md
origin: greenfield
extracted_from: null
subsystem: "SS-01"
capability: "CAP-043"
lifecycle_status: draft
introduced: v1.0-brownfield-backfill
modified: []
deprecated: null
deprecated_by: null
replacement: null
retired: null
removed: null
removal_reason: null
---

# BC-1.18.006: Roll-Before-Write via Block-and-Retry (Not Transparent Redirection) Plus Same-Invocation Atomic Shard-Index Publication

## Description

When BC-1.18.005's size-trigger fires, the dispatcher performs the roll (publish a sealed shard
copy of the current content as a NEW file, then atomically REPLACE the canonical file's content
with empty — see the CORRECTED mechanism below — then atomically publish the updated shard index)
and THEN returns `HookResult::Block` with an explicit, actionable retry instruction — never a
silent transparent redirect, which `HookResult`'s three-variant contract (`Continue`/
`Block { reason }`/`Error { message }`) makes structurally impossible. The blocked call is never
applied, so no shard is ever observed over cap by any downstream reader, and the sealed shard plus
its index update land in the SAME native-gate invocation, guaranteeing they are staged in the same
subsequent factory-artifacts commit (TD-VSDD-053 alignment).

**CORRECTED (fix-burst pass-2, F-P2-003, HIGH) — the seal step is COPY-then-ATOMIC-TRUNCATE-
IN-PLACE, NEVER a rename of the canonical path away.** The v1.0/v1.1 text above ("seal by rename")
described the seal as `rename(canonical, sealed)` followed by a separate `create(canonical)` — two
distinct filesystem operations with an interstitial window, between the rename completing and the
fresh-file create completing, during which the canonical path DOES NOT EXIST ON DISK AT ALL. Any
shard-unaware reader (the ~76 fail-open production plugins with directory-scoped `path_allow`
globs, `check_d_chain_currency`, a human `cat`) that happens to `open()` the canonical path inside
that window observes `ENOENT` — a hard failure, not a stale-but-valid read — directly contradicting
this BC's own AC-007-derived "zero-code-change transparency" guarantee and this BC's own Invariant
3 text ("the canonical filename is NEVER renamed away; only its CONTENT is replaced"), which the
withdrawn rename-based mechanism structurally could not satisfy. See Postcondition 1's corrected
text below for the exact replacement sequence, which reuses ONLY the already-established
`write_atomic` (`crates/last-amended-migrate/src/atomic_write.rs`) temp-file-then-rename primitive
— no new atomic-write primitive, no reimplementation.

## Preconditions

1. BC-1.18.005's size-trigger has determined `projected_size > shard_cap_bytes` for a matched
   `Edit`/`Write`/`MultiEdit` tool call.
2. The current shard file for the matched artifact exists (or is treated as a zero-byte current
   shard per BC-1.18.005 EC-004 if this is the artifact's first-ever write).
3. `crates/hook-sdk/src/result.rs`'s `HookResult` enum exposes exactly three variants (`Continue`,
   `Block { reason }`, `Error { message }`) with no redirect or tool-input-mutation capability —
   this precondition is a structural SDK fact, not a runtime state, and is what makes this BC's
   block-and-retry design the ONLY implementable option for roll-before-write under the current
   dispatcher contract.

4. **NEW (F2 spec-evolution, closing BC-1.18.005 v1.11 Postcondition 3's deferred `replace_all: true`
   occurrence-multiplicity gap; S2502-CLUSTER1-PASS5 STATE.md Drift Item).** BC-1.18.005's
   Postcondition 3 `Edit`/`MultiEdit` formula computes `net_delta_bytes` as a SINGLE-occurrence
   `len(new_string) - len(old_string)` and does not multiply by
   `occurrence_count(old_string, current_file_content)` for a `replace_all: true` call — an
   explicit, adjudicated, DEFERRED gap (BC-1.18.005 Postcondition 3's "Known formula gap"
   sub-paragraph) that this BC MUST NOT treat its roll/block outcome as production-ready against
   until closed. This BC closes the gap WITHOUT modifying BC-1.18.005's already-ACTIVE, already-
   shipped trigger contract — it adopts Option (b) of BC-1.18.005's own pre-authorized closure fork
   ("BC-1.18.006's own roll-execution path... independently re-validates the post-apply size...
   using the ACTUAL applied content length, catching an under-counted trigger before or immediately
   after write"), not Option (a) (amending BC-1.18.005's formula itself). BC-1.18.005's PreToolUse
   single-occurrence estimate remains EXACTLY as BC-1.18.005 specifies it, unchanged; this BC
   instead adds an independent, narrowly-scoped POST-WRITE reconciliation check (Postcondition 7)
   that catches and corrects any resulting under-projection using the artifact's ACTUAL on-disk
   size — no occurrence counting is performed anywhere by this BC. This closure scope applies ONLY
   to an `Edit` call carrying `replace_all: true` and to a `MultiEdit` call containing at least one
   edit block with `replace_all: true`; a plain `Edit`/`MultiEdit`/`Write` without `replace_all:
   true` is unaffected by this Precondition or by Postcondition 7 (BC-1.18.005's formula is already
   exact for those cases, per its Postcondition 3's own UNCHANGED-leg text).

## Postconditions

1. **CORRECTED (fix-burst pass-2, F-P2-003/F-P2-004, HIGH/MEDIUM) — the roll sequence is a
   STAGED, four-step, crash-recoverable operation that executes BEFORE the block is returned:**
   (a) **read** the canonical file's current full content (a one-time, roll-only read — the cheap
   per-write TRIGGER check, BC-1.18.005 Postcondition 2, remains `stat()`-only; content is read
   ONLY once a roll is already confirmed necessary); (b) **publish the sealed shard as a brand-NEW
   file** at `<stem>.<seq:04>.md` (e.g. `decision-log.0001.md`) via `write_atomic` (a `rename()`
   that CREATES a not-yet-existing destination — never interrupts any reader of the canonical
   path, since sealed filenames are never read by shard-unaware code); (c) **atomically REPLACE
   the canonical file's content with empty**, via the SAME `write_atomic` temp-file-then-rename
   primitive — write an empty temp file, then `rename(temp, canonical)`, which is an atomic
   directory-entry REPLACEMENT of an EXISTING destination (POSIX `rename(2)`; the dispatcher's
   Windows target uses `MoveFileEx` with `MOVEFILE_REPLACE_EXISTING`), never a delete-then-create
   — the canonical path resolves to SOME valid file (old content, then instantaneously the new
   empty content) at every observable instant, never absent; (d) **atomically publish the updated
   shard-index TOML** (temp-file-then-rename, the same pattern already established by
   `write_indeterminate_marker` and `write_atomic` — no new atomic-write primitive is introduced).
   Only after (a)-(d) complete does the gate return `HookResult::Block`. **This WITHDRAWS the
   v1.0/v1.1 "seal by rename-away, then create a fresh file" mechanism**, which opened a real
   ENOENT window between the rename and the create (see Description above) — step (b) is a
   `rename()` that CREATES a new sealed path (never vacates the canonical one), and step (c) is a
   `rename()` ONTO the existing canonical path (an atomic replace-in-place), so the canonical path
   is NEVER, at any instant, absent from disk.

   **Partial-failure postconditions — one named `E-SHD-NNN` code per crash point (ADR-051
   Decision 11), because this composite three-write operation (steps b/c/d) has THREE distinct
   crash points, not one:**
   - **Steps (a)-(b) fail (`E-SHD-001`, description REFINED from "seal-rename failure" to
     "shard-seal-write failure" to match the corrected copy-based mechanism — the CODE and
     observable contract are unchanged: `HookResult::Error`, canonical file completely
     untouched):** the canonical file is left in its exact pre-roll state (still over cap, still
     holding its full original content) — safe, no data loss, no duplicate; the next dispatch
     attempt re-evaluates the trigger and re-attempts the FULL sequence from step (a).
   - **Step (c) fails after step (b) succeeded (NEW `E-SHD-006`):** the sealed shard now durably
     exists (a byte-for-byte copy of the pre-roll content) AND the canonical file STILL holds that
     same content too (not yet truncated) — a transient, DETECTABLE duplicate-content state, not a
     data-loss state. **Recovery (self-healing, no operator intervention):** on the NEXT dispatch
     attempt for this artifact, BEFORE evaluating any new trigger, the gate checks whether a
     sealed shard exists at the index's next-expected `seq` path whose content is byte-identical
     to the canonical file's CURRENT content; if so, this is recognized as "seal published,
     truncate did not," and the gate resumes from step (c) alone (re-attempting ONLY the truncate
     + index publish, never re-writing the already-correct sealed shard) — idempotent by
     construction, since step (b)'s `write_atomic` create is itself a no-op if reissued against
     identical content.
   - **Step (d) fails after step (c) succeeded (NEW `E-SHD-007`):** the canonical file is
     CORRECTLY fresh and empty (safe for all future writes — no over-cap risk, no data loss) and
     the sealed shard file exists correctly on disk, but `<artifact-stem>.shard-index.toml` has
     not yet recorded the new `[[shard]]` entry — a discoverability-METADATA gap only:
     whole-corpus glob-based readers (`<stem>*.md`) still find the sealed file regardless of index
     membership, so no reader-visible data loss occurs. **Recovery (self-healing):** on the next
     dispatch attempt, the gate reconciles the index by scanning the filesystem for sealed-shard
     files matching the artifact's naming convention that are absent from the index, and appends
     the missing entries before evaluating any new trigger.

     **CORRECTED (cluster-2 LOCAL adversary pass-1, F-C2-P1-001, MAJOR) — the reconciled entry MUST
     recover `sealed_retroactively` by inference, never hardcode it `false`.** Postcondition 5
     designates `sealed_retroactively` the SOLE audit trail distinguishing a legitimately-over-cap
     sealed shard from one guaranteed `<= shard_cap_bytes`. If the orphaned entry belonged to a
     RETROACTIVE roll (Postcondition 7 catch point (i) or (ii), which crashed after its OWN step (c)
     but before its OWN step (d)), a filesystem-only reconciliation that is blind to which roll
     produced the file and defaults the field to `false` would mislabel a genuinely over-cap shard
     as cap-guaranteed — the exact ambiguity Postcondition 5 exists to prevent, and the only record
     of the roll's retroactiveness (the in-flight roll's own knowledge of which path invoked it) is
     lost the moment step (d) fails to durably record it. The value is, however, deterministically
     INFERRABLE from data already available to the reconciliation scan, with no in-memory roll
     context required: `bytes_at_seal > shard_cap_bytes ⇒ the seal must have been retroactive`,
     because a PROSPECTIVE (Postcondition 1, non-retroactive) roll can never seal over-cap content —
     Postcondition 3's unconditional guarantee ensures every prospectively-sealed shard's
     `bytes_at_seal <= shard_cap_bytes`, while only a retroactive roll's documented exception
     (Postcondition 7's "Sealed-shard cap exception") permits `bytes_at_seal > shard_cap_bytes`. This
     BC therefore REQUIRES: when reconciling an orphaned, un-indexed sealed shard, the gate MUST
     compute `bytes_at_seal` from the sealed file's actual on-disk size (already required to
     populate the recovered `[[shard]]` entry's `bytes_at_seal` field) and set
     `sealed_retroactively = true` if `bytes_at_seal > shard_cap_bytes`, else `false` (or omit, per
     Postcondition 5's default) — never a hardcoded `false` regardless of size. See Invariant 7 for
     the general form of this inference rule.
   - **All four steps succeed:** normal `Block` outcome (Postcondition 2), no error.

2. **CORRECTED (fix-burst pass-2, F-P2-002/F-P2-003, HIGH) — the observable outcome of an over-cap
   write is `HookResult::Block` with a specific, actionable, UNIFIED retry-instruction message —
   NOT a silent transparent redirect, and NOT a tool-divergent instruction.** The block reason
   text MUST include: the artifact name, the cap that was reached (in bytes), the fact that the
   current shard is now empty, and a SINGLE unified retry instruction (the SAME wording regardless
   of the original tool, since both branches now converge on "recompute against the current,
   post-roll state"):

   > "Shard `<artifact>` rotated (cap `<N>` bytes reached); the current shard is now empty. Retry
   > your write against the CURRENT (post-roll, empty) file — do not resubmit your original
   > payload unchanged: if you used `Edit` or `MultiEdit`, your `old_string` will no longer match
   > (the content it targeted is now in `<sealed-path>`) — reissue as a fresh `Write` containing
   > ONLY your new entry; if you used `Write`, recompute `content` to contain ONLY your new entry
   > (not your original full pre-roll payload, which reflects discarded state and will exceed the
   > cap again if resubmitted)."

   **This WITHDRAWS the v1.0/v1.1 "if you used `Write`, simply retry unchanged" wording**, which
   was UNSOUND under BOTH the withdrawn rename mechanism and the corrected per-tool
   `projected_size` formula (BC-1.18.005 Postcondition 3, F-P2-002): because the canonical file is
   now EMPTY after a roll, and because a blocked `Write`'s own `content` parameter was composed by
   the agent BEFORE the roll (typically by reading the OLD, over-cap file and appending one new
   entry), retrying that SAME `content` unchanged would resubmit content that is STILL over cap
   relative to the fresh empty shard (since `projected_size = len(content)` for `Write`, per the
   corrected formula, and `len(content)` has not shrunk) — producing a permanent block/retry
   deadlock, not a duplicate. This is a structural consequence of `HookResult`'s three-variant
   contract (Precondition 3): no design that assumes transparent write-redirection is
   implementable, so the postcondition describes the message an agent WILL see, not a hypothetical
   silent success.

3. **No shard is ever observed in an over-cap state by any downstream reader.** Because the seal
   happens before the block (Postcondition 1), and the blocked call is never applied to any file,
   the sealed shard's final size is always `<= shard_cap_bytes` (BC-1.18.005's cap, evaluated at
   seal time) and the new current shard starts at exactly 0 bytes. This is a structural guarantee,
   not a convention: it holds even if the agent never retries (the sealed shard is already
   durably capped; only the RETRY's content is lost if the agent abandons the operation).

4. **Shard index and sealed shard land in the same native-gate invocation.** The shard-index TOML
   write (Postcondition 1 step (d)) and the sealed-shard publish (step (b)) are both filesystem
   writes issued by the SAME PreToolUse invocation, before any `git add`/`git commit` occurs downstream.
   This makes TD-VSDD-053's single-commit-per-burst hold STRUCTURALLY for shard+index atomicity
   (not merely by state-manager discipline), because both writes are guaranteed to be present in
   the working tree before the next `git commit` regardless of which agent or skill issued the
   original tool call.

5. **Shard-index schema (one file per sharded mechanism-A artifact):**
   ```toml
   # .factory/cycles/<cycle>/<artifact-stem>.shard-index.toml
   schema_version = 1
   artifact_stem = "decision-log"
   current_shard = "decision-log.md"
   shard_cap_bytes = 49152           # calibrated per BC-1.18.005; locked at F4
   max_single_record_bytes = 16384
   safety_margin_bytes = 8192
   practical_fuel_ceiling = 8000000
   worst_case_fuel_per_byte = 106.36

   [[shard]]
   seq = 1
   path = "decision-log.0001.md"
   sealed_at = "2026-09-10T00:00:00Z"
   bytes_at_seal = 49087
   ```
   Every seal event appends exactly one new `[[shard]]` table entry with `seq` incrementing
   monotonically from 1, `path` naming the sealed file, `sealed_at` in UTC ISO-8601, and
   `bytes_at_seal` recording the sealed shard's exact final byte count (always `<= shard_cap_bytes`
   per Postcondition 3).

   **NEW field (F2 spec-evolution, replace_all closure) — `sealed_retroactively` (boolean, OPTIONAL,
   default `false` when omitted — backward compatible with every `[[shard]]` entry produced before
   Postcondition 7 existed).** A seal produced by Postcondition 7's retroactive reconciliation path
   (either catch point (i) or catch point (ii)) MUST set `sealed_retroactively = true`; every seal
   produced by Postcondition 1's normal pre-write block-and-retry path MUST omit the field or set it
   `false`. This is the sole audit trail distinguishing a shard whose `bytes_at_seal` may
   legitimately exceed `shard_cap_bytes` (Postcondition 7's documented, narrowly-scoped exception)
   from one that is guaranteed `<= shard_cap_bytes` (Postcondition 3's normal, unconditional
   guarantee). **CORRECTED (cluster-2 LOCAL adversary pass-1, F-C2-P1-001, MAJOR):** when this field
   must be RECOVERED during self-heal reconciliation of an orphaned, un-indexed sealed shard
   (Postcondition 1's `E-SHD-007` recovery path), the gate MUST NOT default it to `false`
   unconditionally — it MUST apply the deterministic inference rule specified in Invariant 7
   (`bytes_at_seal > shard_cap_bytes ⇒ sealed_retroactively = true`).

6. **Stable-current-filename addressing is a consequence of this BC's seal mechanism, not a
   separate lookup step.** Because the seal PUBLISHES a copy of the old content as a NEW sealed
   file and ATOMICALLY REPLACES the canonical file's content with empty (CORRECTED, F-P2-003 —
   never renames the canonical path away), any shard-unaware reader or validator that opens the
   canonical filename always sees the current/latest shard, with zero code change required on the
   reader's part. Whole-corpus readers use the glob `<stem>*.md` (e.g. `decision-log*.md`), which
   sorts sealed shards ascending before the current file — the deciding byte-comparison one
   position past the shared `<stem>` prefix is a digit (`0`-`9`, from a sealed shard's `.NNNN.md`
   suffix) vs. `m` (from the current file's own `.md` suffix); since every digit byte is
   numerically less than `m`, sealed shards sort first (ADR-051 §Decision 3 fix-burst-corrected
   sort-order rationale — the "current file sorts last" conclusion is unchanged) — no
   special-casing needed in a `sort`-fed pipeline.

7. **NEW (F2 spec-evolution, closing BC-1.18.005 v1.11 Postcondition 3's deferred `replace_all:
   true` occurrence-multiplicity gap, S2502-CLUSTER1-PASS5) — bounded post-write reconciliation for
   `replace_all: true` `Edit`/`MultiEdit` calls.**

   **Why a post-write check, not a pre-write formula fix.** BC-1.18.005's Postcondition 3 PreToolUse
   trigger is, by Precondition 4 above, left unchanged: it evaluates `projected_size` using the
   single-occurrence delta BEFORE the write is applied. When the call carries `replace_all: true`
   and the TRUE occurrence-multiplied delta would have pushed `projected_size` over
   `shard_cap_bytes` while the single-occurrence estimate did not, the trigger returns `Continue`
   and the tool call is applied normally by the underlying editor — this BC's own block-and-retry
   roll (Postcondition 1) is a downstream CONSEQUENCE of BC-1.18.005's trigger firing (Precondition
   1) and is therefore structurally never invoked for a call the trigger itself failed to flag.
   Correcting this requires either changing BC-1.18.005's own trigger formula (Option (a) of
   BC-1.18.005's pre-authorized closure fork — amending an ALREADY-ACTIVE, already-shipped BC's
   contract, NOT taken here) or catching the resulting under-count independently, after the fact,
   using the artifact's true on-disk state (Option (b), specified below). Option (b) needs NO
   occurrence counting at all: a `stat()` of the ACTUAL post-apply file gives the TRUE size
   directly, exactly as it will for the artifact's own next trigger evaluation; no
   `len(new_string)`/`len(old_string)`/occurrence-count arithmetic is required or performed by this
   Postcondition.

   **Scope — catch point (i) fires only for the narrow `replace_all` case; catch point (ii)
   backstops EVERY subsequent matched dispatch, INCLUDING `Write` (CORRECTED, cluster-2 LOCAL
   adversary pass-1, F-C2-P1-002).** Catch point (i) (the immediate post-write reconciliation
   check) runs ONLY for an `Edit` call whose `replace_all` field is `true`, or a `MultiEdit` call
   containing at least one edit block whose `replace_all` field is `true`, AND whose target path
   matches a `[[shard]]` config entry (the SAME match BC-1.18.005 Precondition 3/Postcondition 1
   already performs — no additional config lookup); a plain `Edit`/`MultiEdit` without
   `replace_all: true`, a `Write`, or any call against an unmatched path never triggers catch point
   (i) itself, and pays zero added cost from it. Catch point (ii), by contrast, MUST run before
   EVERY subsequent matched dispatch — `Edit`, `Write`, or `MultiEdit` alike — because its job is
   to detect a crash-orphaned over-cap canonical file left behind by ANY failure of catch point (i),
   regardless of what tool the NEXT call happens to use; scoping catch point (ii) to `replace_all`
   calls only would leave exactly the data-loss gap F-C2-P1-002 identified (an intervening `Write`
   could destroy the orphaned history before a `replace_all` call ever came along to trigger a
   check). The withdrawn "a `Write` is completely unaffected" claim applied correctly to catch
   point (i) alone but was over-generalized to imply Postcondition 7 as a whole imposes zero cost on
   `Write` — corrected: `Write` pays one dedicated, bounded `stat()` per matched dispatch for catch
   point (ii) (see catch point (ii) above), while remaining entirely exempt from catch point (i).

   **Mechanism — two redundant catch points, bounding the observable-over-cap window to at most one
   subsequent matched dispatch:**
   - **(i) Immediate post-write check (primary).** In the SAME native dispatcher handling that
     already distinguishes PreToolUse (BC-1.18.005 Precondition 1) from PostToolUse for this tool
     call, once a `replace_all: true` call has been applied, this BC's gate performs a fresh
     `stat()` of the canonical file. If `actual_size <= shard_cap_bytes`, no action is taken (the
     single-occurrence estimate was conservative or exactly correct; `Continue`'s outcome stands
     unmodified). If `actual_size > shard_cap_bytes`, this BC executes Postcondition 1's EXACT
     four-step sequence (read the now-over-cap canonical content, publish it as a sealed shard,
     atomically truncate the canonical file to empty, publish the updated shard-index)
     RETROACTIVELY — i.e., against content that is ALREADY on disk, not content about to be
     written. The three per-step partial-failure codes (`E-SHD-001`/`E-SHD-006`/`E-SHD-007`) apply
     identically, self-healing exactly as Postcondition 1 already specifies; no new error code is
     introduced for the retroactive invocation itself.
   - **(ii) Next-dispatch backstop (defense-in-depth, covers a crash between apply and (i)).**
     Before evaluating BC-1.18.005's own trigger for ANY subsequent `Edit`, `Write`, or `MultiEdit`
     against a matched artifact, this BC's gate first checks whether the CURRENT on-disk size
     already exceeds `shard_cap_bytes` — a state that is structurally impossible to persist under
     the normal pre-write block-and-retry flow, and should already have been resolved by (i), but
     can transiently survive a dispatcher crash between the `replace_all` write's completion and
     (i)'s own execution. If so, this BC executes the SAME retroactive four-step sequence BEFORE
     applying the new call or evaluating its own trigger.

     **CORRECTED (cluster-2 LOCAL adversary pass-1, F-C2-P1-002, MAJOR) — the backstop's probe cost
     is per-tool, not uniformly free; a `Write` requires a DEDICATED stat, not a reused one.** The
     prior text above claimed the backstop reuses "the same `stat()` BC-1.18.005 Postcondition 2
     already performs" for EVERY subsequent tool call. This is TRUE for `Edit`/`MultiEdit` — whose
     Postcondition 3 formula, `projected_size = current_shard_bytes + net_delta_bytes`, already
     requires `stat()`-reading `current_shard_bytes`; the backstop reuses that already-read value,
     adding zero new `stat()` calls. It was WRONG for `Write`: `Write`'s Postcondition 3 formula is
     `projected_size = len(content)` alone, with NO `stat()` of the canonical file anywhere in
     BC-1.18.005's own `Write` trigger path — there is no existing stat-read for this backstop to
     reuse. Left uncorrected, this was a real DATA-LOSS gap, not merely a wording inaccuracy: after a
     dispatcher crash leaves the canonical file over-cap and un-sealed (EC-015), a subsequent `Write`
     whose own `content` is under-cap would apply directly against the canonical file — REPLACING,
     and permanently DESTROYING, the un-sealed over-cap history — with no roll ever having started
     for self-heal to catch (self-heal reconciles orphaned ROLLS; it has nothing to reconcile if no
     roll was ever attempted). Under CLAUDE.md's production-grade default (no version of this gate
     may lose history), this backstop therefore performs its OWN dedicated, bounded `stat()` of the
     canonical file BEFORE applying a `Write` — an ADDITIONAL probe specific to the `Write` arm, NOT
     a change to BC-1.18.005's stat-free `Write` trigger formula (which is UNCHANGED: this dedicated
     stat feeds ONLY this backstop's over-cap check, never `projected_size`). For `Edit`/`MultiEdit`,
     the backstop remains cost-free (reuses the existing stat), unchanged from the original design
     intent. Net cost: one bounded `stat()` call added per `Write` against a matched artifact
     (independent of artifact size, since `stat()` never reads file content); zero added cost for
     `Edit`/`MultiEdit`.

   **Bounded-window postcondition (testable, not "known limitation" prose).** For a `replace_all:
   true` call whose true occurrence-multiplied delta was under-projected by BC-1.18.005's
   single-occurrence trigger estimate, the canonical file MAY be observed in an over-cap state ONLY
   during the window between that call's completed write and the EARLIER of: (i) firing for the
   SAME tool invocation, or (ii) the artifact's NEXT matched `Edit`/`Write`/`MultiEdit` dispatch
   (whichever occurs first) — never indefinitely, never spanning more than one subsequent matched
   dispatch. This is the SOLE, explicitly bounded exception to Postcondition 3's "no shard is ever
   observed in an over-cap state by any downstream reader" structural guarantee, and it applies
   ONLY to this narrow `replace_all: true` case — every other call class this BC governs retains
   Postcondition 3's unconditional guarantee.

   **Sealed-shard cap exception (documented, scoped only to retroactive rolls).** Because a
   retroactive roll seals content that has ALREADY been written (its size could not be prevented
   pre-write), the resulting sealed shard's `bytes_at_seal` MAY exceed `shard_cap_bytes` — a
   documented, narrow exception to Postcondition 3's per-shard cap guarantee, scoped ONLY to shards
   produced by this Postcondition (marked `sealed_retroactively: true`, Postcondition 5). The
   CANONICAL file's own guarantee is UNAFFECTED and holds unconditionally once reconciliation
   completes: step (c)'s atomic truncate-to-empty never accepts an exception, so the canonical file
   is always exactly 0 bytes immediately after either catch point (i) or (ii) fires — see
   Invariant 6.

## Invariants

1. **`HookResult::Block` is the ONLY variant this BC's gate returns on an over-cap write.** No
   version of this check may return `Continue` after determining `projected_size >
   shard_cap_bytes` (that would silently permit an over-cap write, violating BC-1.18.005
   Postcondition 3's contract), and no version may return `Error` for a normal (non-crash)
   over-cap condition (a normal rotation is not an error condition — it is the mechanism working
   as designed).

2. **CORRECTED (fix-burst pass-2, F-P2-003, HIGH) — the read-publish-truncate-publish sequence is
   never reordered.** Publishing the shard index before the sealed shard is published, or
   truncating the canonical file's content before the sealed shard's content is durably published,
   would risk a window where NEITHER a valid sealed copy NOR a valid canonical-with-full-content
   state exists for the pre-roll content. The four sub-steps in Postcondition 1 ((a) read, (b)
   publish sealed shard, (c) atomic-truncate canonical, (d) publish index) execute in the stated
   order; each of (b), (c), and (d) is its own independent `write_atomic` temp-file-then-rename
   call — there is no OS-level atomicity spanning multiple steps, which is exactly why
   Postcondition 1's per-step partial-failure codes (`E-SHD-001`/`E-SHD-006`/`E-SHD-007`) exist:
   a crash between any two steps is a distinct, named, self-healing-recoverable state, never an
   unspecified one.

3. **The canonical filename never moves.** At every observable point in time — before a roll,
   during a roll's execution, and after a roll completes — the canonical filename (e.g.
   `decision-log.md`) refers to SOME valid file: either the not-yet-full current shard (before a
   roll), or the fresh empty current shard (after a roll completes). It is NEVER renamed away to a
   sealed name; only its CONTENT is replaced, via an atomic `write_atomic` rename-ONTO-existing-
   destination (Postcondition 1 step (c)) — never a rename-OUT-of the canonical path. This
   invariant is now structurally, not merely conventionally, true: the withdrawn v1.0/v1.1
   rename-away mechanism could not satisfy it (see Description's ENOENT-window analysis); the
   corrected copy-then-atomic-truncate mechanism can, because `rename()` onto an EXISTING
   destination never leaves that destination absent.

4. **Retry-instruction wording is a single, fixed template — never divergent per original tool
   name.** CORRECTED (fix-burst pass-2, F-P2-002): the withdrawn v1.0/v1.1 design chose between
   two DIFFERENT wordings based on the original blocked tool's name (a `Write`-specific "simply
   retry unchanged" branch that was later found unsound). The corrected design (Postcondition 2)
   uses ONE unified message template that names both tool cases within the SAME text — the
   template itself never varies, and its content is never randomized, never omitted, and never
   generic ("write failed, try again" without the specific per-tool guidance embedded in the
   unified template is insufficient).

5. **NEW (fix-burst pass-2, F-P2-005, MEDIUM) — the append-only-tail assumption is explicit,
   never silently relied upon.** This BC's gate has NO semantic understanding of WHERE within a
   file an `Edit`/`Write`/`MultiEdit` lands — it computes `projected_size` from a pure byte-delta/
   length formula (BC-1.18.005 Postcondition 3) only. The roll+block+retry wording (Postcondition
   2) is phrased for the common case this gate exists to serve: a pure APPEND of one new record at
   the file's end. The four mechanism-A artifacts are, by construction, POLICY-1
   (`append_only_numbering`) governed append-only records — POLICY-1 already forbids renumbering
   or rewriting historical entries, so legitimate `Edit`/`MultiEdit` mutations against these
   artifacts are, by that SAME policy, already expected to be either (a) a pure append of a
   brand-new record at file end, or (b) a narrow amendment to a STILL-MUTABLE, recently-added
   record near the tail — never an edit to arbitrarily old, already-sealed, or deep-mid-file
   historical content. See EC-012 below for the caller-responsibility failure mode when this
   assumption is violated, and its sanctioned escape hatch.

6. **NEW (F2 spec-evolution, replace_all closure) — the canonical file's zero-bytes-after-roll
   guarantee holds unconditionally, even under Postcondition 7's retroactive path; only the SEALED
   shard's per-shard cap guarantee is exceptionally relaxed, and only when `sealed_retroactively:
   true`.** Postcondition 1 step (c) (atomic truncate-to-empty) is IDENTICAL code whether invoked
   from the normal pre-write block-and-retry sequence or from Postcondition 7's retroactive
   reconciliation (catch point (i) or (ii)) — there is no code path in which the canonical file is
   left non-empty after either catch point completes successfully. Invariant 3's "canonical filename
   never moves" guarantee is likewise unaffected: Postcondition 7 reuses the exact same
   rename-ONTO-existing-destination truncate, never a rename-away.

7. **NEW (cluster-2 LOCAL adversary pass-1, F-C2-P1-001, MAJOR) — `sealed_retroactively` is
   deterministically inferrable from `bytes_at_seal`, and self-heal reconciliation MUST use that
   inference, never a hardcoded default.** For any `[[shard]]` entry, `bytes_at_seal >
   shard_cap_bytes` (the value recorded in that SAME entry) if and only if the seal was produced by
   Postcondition 7's retroactive path — a PROSPECTIVE roll (Postcondition 1) can never seal
   over-cap content (Postcondition 3's unconditional per-shard cap guarantee), so `bytes_at_seal >
   shard_cap_bytes` occurring at all is proof-by-construction of retroactivity; conversely, a
   RETROACTIVE roll seals content that has already been written and whose size could not be
   prevented pre-write, so `bytes_at_seal <= shard_cap_bytes` from a retroactive roll is possible
   (the under-projection may have been small) but never contradicts the inference in the direction
   that matters — the inference is used only in the `>` case, which is unambiguous. Postcondition
   1's `E-SHD-007` self-heal reconciliation (which appends an index entry for a sealed shard found
   on disk but missing from the index, with no other context available about which roll produced
   it) MUST apply this inference — `sealed_retroactively = (bytes_at_seal > shard_cap_bytes)` —
   rather than defaulting the recovered entry's `sealed_retroactively` to `false` unconditionally.
   This closes the mislabeling gap that would otherwise arise when a retroactive roll's own step
   (d) fails (`E-SHD-007`) before the roll's own retroactive-context could be durably recorded.

## Edge Cases

| ID | Description | Expected Behavior |
|----|-------------|-------------------|
| EC-001 | Single appended block exceeds shard cap (story's own EC-001) | Roll to new shard BEFORE the write is applied (Postcondition 1); the write itself is blocked (Postcondition 2), never partially applied to either the sealed or fresh shard |
| EC-002 | Agent issues `Edit` against a just-rolled artifact without reading the block message | `Edit`'s `old_string` will not match the now-empty fresh shard — the `Edit` tool call itself fails with a standard "old_string not found" error; this is the exact confusing-failure class Postcondition 2's explicit retry wording exists to prevent for the FIRST block, but a repeated blind-retry `Edit` after that still fails at the tool layer (this BC's contract covers the dispatcher's OWN block message, not enforcement of agent compliance) |
| EC-003 | Postcondition 1 step (a)-(b) fails (e.g., filesystem permission error, disk full) before the sealed shard is durably published | Fail-loud: the gate returns `HookResult::Error` (`E-SHD-001`, description refined to "shard-seal-write failure"), not `Block` and not `Continue` — the canonical file is left in its exact pre-roll state; see EC-005 of the S-25.02 story draft ("Shard index unavailable or corrupt") for the sibling dispatch-blocked case |
| EC-004 | Concurrent dispatch attempts to write to the same artifact's shard (two agents/sessions racing) | Atomic temp+rename for the index publish ensures no torn shard-index write; TD-VSDD-053 single-commit-per-burst and the project's factory-lock discipline (ADR-025) prevent concurrent factory-artifacts commits from landing interleaved |
| EC-005 | `MultiEdit` with one edit block that alone exceeds the cap even against a freshly-rolled (0-byte) shard | Roll triggers as normal (BC-1.18.005 EC-004: current_shard_bytes=0), but if `payload_bytes` alone exceeds `shard_cap_bytes - MAX_SINGLE_RECORD_BYTES`'s margin, this indicates a single record larger than the calibrated `MAX_SINGLE_RECORD_BYTES` assumption — the write still blocks per Postcondition 2, and the retry-then-still-too-large condition surfaces to the agent as a repeated block, which is the correct fail-loud signal that the record itself needs to be split by the caller, not silently truncated |
| EC-006 | `/compact-state`'s own `Edit`/`Write` calls against a sharded artifact trigger a mid-extraction roll | Gets shard-awareness for free (ADR-051 Decision 5) — the skill receives the same `Block`-with-retry-instruction message any other caller would; no amendment to the gate mechanism itself is required for this (a small documentation-only note to `compact-state/SKILL.md`'s own retry-loop guidance is recommended, per ADR-051 Decision 5, but is out of this BC's and this burst's write scope — SKILL.md is not a `.factory/specs/` artifact) |
| EC-010 (fix-burst pass-2, F-P2-004) | Postcondition 1 step (c) (atomic-truncate) fails AFTER step (b) (sealed-shard publish) succeeded | `E-SHD-006`: sealed shard durably exists AND canonical file still holds the same pre-roll content (transient, detectable duplicate-content state, not data loss); self-healing recovery resumes from step (c) alone on the next dispatch attempt (Postcondition 1) |
| EC-011 (fix-burst pass-2, F-P2-004) | Postcondition 1 step (d) (index publish) fails AFTER step (c) (atomic-truncate) succeeded | `E-SHD-007`: canonical file is correctly fresh/empty and the sealed shard exists correctly on disk, but the shard-index has not yet recorded the new `[[shard]]` entry (discoverability-metadata gap only, no reader-visible data loss); self-healing index reconciliation runs on the next dispatch attempt (Postcondition 1) |
| EC-012 (fix-burst pass-2, F-P2-005) | An `Edit`/`MultiEdit`'s target content was ALREADY relocated to a SEALED shard by an EARLIER roll (a policy-violating attempt to amend deep-historical content, or a caller operating on stale in-memory state) | The retry-instruction text ("reissue as a fresh `Write` containing only your new entry") is INAPPLICABLE — there is no "new entry" to reissue against the canonical file, because that content no longer lives there. **Sanctioned escape hatch:** a sealed shard file (`<stem>.<seq:04>.md`) is an ORDINARY file that does NOT match any `[[shard]]` config entry's canonical-path pattern (BC-1.18.005 Postcondition 1's zero-cost bypass for unmatched paths) — it is entirely UNGATED by this BC's gate, and a caller with a genuine, policy-sanctioned need to touch historical content addresses the sealed file DIRECTLY by its own on-disk filename, exactly as it would edit any other ordinary file. This gate makes no attempt to detect, permit, or forbid such an edit — that is POLICY-1's concern (enforced at the `consistency-validator`/adversary-prompt agent level), entirely orthogonal to this gate's byte-size-triggered rotation concern. |
| EC-013 (fix-burst pass-2, F-P2-005) | A net-positive `Edit`/`MultiEdit` targets a STILL-MUTABLE tail record (e.g., a same-burst typo fix to an entry not yet sealed away) and pushes `projected_size` over cap | Triggers the SAME generic roll+block+retry sequence as any other over-cap write (Postcondition 1/2) — Invariant 5's append-only-tail assumption is not violated by this case (the target is still-mutable tail content, not deep-historical content); EC-002's `old_string`-mismatch failure mode applies identically |
| EC-014 (NEW, F2 spec-evolution, replace_all closure) | An `Edit{replace_all: true}` against a matched artifact has multiple occurrences of `old_string`; BC-1.18.005's single-occurrence estimate keeps `projected_size <= shard_cap_bytes`, but the TRUE occurrence-multiplied delta would have exceeded it | BC-1.18.005's trigger returns `Continue` (under-projected); the `Edit` is applied, producing an actual over-cap canonical file; Postcondition 7 catch point (i) fires immediately post-write, detects `actual_size > shard_cap_bytes` via `stat()`, and executes the retroactive roll: canonical truncated to 0 bytes, sealed shard published with `bytes_at_seal` equal to the TRUE (over-cap) size and `sealed_retroactively: true` (Postcondition 5) |
| EC-015 (NEW, F2 spec-evolution, replace_all closure) | Postcondition 7 catch point (i) fails to execute at all (e.g., dispatcher process crash between the `replace_all` write's completion and (i)'s own invocation) | The canonical file is left over cap on disk with no seal; catch point (ii) — the leading `stat()`-based backstop probe on the SAME artifact's NEXT matched `Edit`/`Write`/`MultiEdit` dispatch — detects the pre-existing over-cap state BEFORE evaluating that new call's own trigger, and executes the same retroactive roll; Postcondition 7's bounded-window guarantee ("at most one subsequent matched dispatch") holds |
| EC-016 (NEW, F2 spec-evolution, replace_all closure) | Agent's `Edit{replace_all: true}` tool call returns SUCCESS (`Continue`, not `Block`) to the agent, and a retroactive roll subsequently occurs via catch point (i) with no block/retry message ever shown to the agent | Expected and unchanged from this BC's existing EC-002 scope ("this BC's contract covers the dispatcher's OWN block message, not enforcement of agent compliance"): the agent has no proactive signal that a roll occurred; its next `Edit` against the (now-empty) canonical file fails at the tool layer with a standard "old_string not found" error, identically to EC-002's already-specified outcome — no new agent-facing contract is introduced by Postcondition 7 |
| EC-017 (NEW, cluster-2 LOCAL adversary pass-1, F-C2-P1-002) | Catch point (i) crashes without running (as EC-015), and the artifact's NEXT matched dispatch is specifically a `Write` (not an `Edit`/`MultiEdit`) whose own `content` is under-cap | Catch point (ii)'s DEDICATED `stat()` of the canonical file (CORRECTED, F-C2-P1-002 — a probe added specifically because `Write`'s own Postcondition-3 trigger formula performs no `stat()` of its own) detects `actual_size > shard_cap_bytes` BEFORE the `Write` is applied, and executes the retroactive four-step roll (sealing the orphaned over-cap content with `sealed_retroactively: true`) BEFORE the `Write`'s under-cap `content` is ever written to the canonical file — closing the data-loss path where an unguarded `Write` would otherwise silently replace and destroy the un-sealed over-cap history |
| EC-018 (NEW, cluster-2 LOCAL adversary pass-1, F-C2-P1-001) | A retroactive roll (Postcondition 7 catch point (i) or (ii)) crashes after its own step (c) but before its own step (d), i.e. `E-SHD-007` occurs for a RETROACTIVE (not prospective) roll | Self-heal reconciliation (Postcondition 1's `E-SHD-007` recovery) infers `sealed_retroactively` from the recovered entry's own `bytes_at_seal` vs. `shard_cap_bytes` (Invariant 7) rather than hardcoding `false`; since the roll was retroactive, `bytes_at_seal > shard_cap_bytes` and the recovered entry correctly sets `sealed_retroactively=true` |

## Canonical Test Vectors

| Input | Expected Output | Category |
|-------|----------------|----------|
| **CORRECTED (cluster-2 LOCAL adversary pass-1, F-C2-P1-004 — replaces a stale carry-over from the WITHDRAWN `current + payload` formula, which this row's prior numeric example silently still assumed).** `Write` to `decision-log.md`, current shard 45,000 bytes (irrelevant to the `Write` trigger per BC-1.18.005 Postcondition 3's `projected_size = len(content)` formula), `content` length 50,000 bytes, cap 49,152 (`50,000 > 49,152` → trigger fires) | Publish `decision-log.0001.md` (a NEW file, byte-copy of the 45,000-byte PRE-ROLL canonical content — the sealed shard's size is always the pre-roll canonical content, never the blocked `Write`'s own `content`, which is never applied); atomically replace `decision-log.md`'s content with empty (0 bytes, canonical path never absent); publish index `[[shard]] seq=1 path="decision-log.0001.md" bytes_at_seal=45000`; return `Block{reason: "Shard decision-log rotated (cap 49152 bytes reached)... recompute content to contain ONLY your new entry..."}` (unified wording) | happy-path |
| **CORRECTED (fix-burst pass-2, F-P2-002).** `Edit` to `burst-log.md` causing an over-cap projection | Roll executes identically to the `Write` case (copy-then-atomic-truncate); `Block` reason text uses the SAME unified template as the `Write` case (naming both tool branches within one message, per Postcondition 2) | happy-path |
| Second roll on the same artifact within the same cycle | `[[shard]]` gains a SECOND entry with `seq=2`; `seq=1`'s entry is untouched (append-only index) | edge-case |
| Postcondition 1 step (a)-(b) fails (simulated permission error before sealed shard is published) | `HookResult::Error` (`E-SHD-001`); no shard-index entry published; canonical file left in its pre-roll state | error |
| **NEW (fix-burst pass-2, F-P2-004).** Postcondition 1 step (c) fails after step (b) succeeded (simulated crash between sealed-shard publish and canonical truncate) | `E-SHD-006`: sealed shard `decision-log.0001.md` exists AND `decision-log.md` still holds the same 45,000-byte content; next dispatch attempt detects the byte-identical duplicate and resumes from step (c) alone (truncate + index publish only) | error |
| **NEW (fix-burst pass-2, F-P2-004).** Postcondition 1 step (d) fails after step (c) succeeded (simulated crash between canonical truncate and index publish) | `E-SHD-007`: `decision-log.md` is correctly empty and `decision-log.0001.md` exists on disk, but the shard-index has no `[[shard]] seq=1` entry; next dispatch attempt scans for un-indexed sealed shards and appends the missing entry before evaluating any new trigger | error |
| Whole-corpus `grep -n "D-1234" decision-log*.md` after 2 rolls | Glob matches `decision-log.0001.md`, `decision-log.0002.md`, `decision-log.md` in that lexicographic (and chronological) order | happy-path |
| **NEW (fix-burst pass-2, F-P2-005).** `Edit` targets content already relocated to `decision-log.0001.md` by an earlier roll | Retry-instruction text is inapplicable against the (now-unrelated) canonical file; caller addresses `decision-log.0001.md` directly by filename — ungated, since it matches no `[[shard]]` config entry | edge-case |
| **NEW (F2 spec-evolution, replace_all closure, EC-014).** `Edit{replace_all: true}` to `decision-log.md`; current shard 48,900 bytes; `old_string` (50 bytes) occurs 3 times; `new_string` 250 bytes (per-occurrence delta +200); single-occurrence projected size = 49,100 (`<= 49,152` cap → BC-1.18.005 trigger returns `Continue`, write applied); TRUE post-apply size = 48,900 + 3×200 = 49,500 (`> 49,152`) | Postcondition 7 catch point (i) `stat()`s the canonical file post-write, detects 49,500 > 49,152, executes retroactive roll: publishes `decision-log.0001.md` (49,500 bytes, `bytes_at_seal=49500`, `sealed_retroactively=true`); atomically truncates `decision-log.md` to 0 bytes; publishes the index entry | edge-case |
| **NEW (F2 spec-evolution, replace_all closure, EC-015).** Same over-cap `replace_all` write as above, but Postcondition 7 catch point (i) is simulated as crashed/never-run | Canonical file remains at 49,500 bytes (over cap) until the artifact's next matched dispatch; catch point (ii)'s leading probe detects `actual_size (49,500) > shard_cap_bytes (49,152)` BEFORE evaluating the new call's own trigger, and executes the retroactive roll identically to the row above | edge-case |
| **NEW (F2 spec-evolution, replace_all closure).** `Edit{replace_all: true}` where `old_string` occurs exactly once (`occurrence_count=1`) | Single-occurrence estimate equals the TRUE delta exactly (the identity case) — BC-1.18.005's trigger is exact; Postcondition 7 catch point (i)'s `stat()` finds `actual_size <= shard_cap_bytes` and takes no action | happy-path |
| **NEW (cluster-2 LOCAL adversary pass-1, F-C2-P1-002, EC-017).** Catch point (i) crashes (as in the EC-015 vector above); canonical file left at 49,500 bytes (over cap, un-sealed, no roll ever started); artifact's NEXT matched dispatch is a `Write` with `content` length 3,000 bytes (under cap on its own) | Catch point (ii) performs its OWN dedicated `stat()` of the canonical file (no reuse of a `Write`-side stat, since `Write`'s trigger performs none) BEFORE applying the `Write`; finds `actual_size (49,500) > shard_cap_bytes (49,152)`; executes the retroactive roll: publishes `decision-log.0001.md` (`bytes_at_seal=49500`, `sealed_retroactively=true`), atomically truncates `decision-log.md` to 0 bytes, publishes the index entry — ONLY THEN is the `Write`'s 3,000-byte `content` applied, against the now-empty canonical file, with no history lost | edge-case |
| **NEW (cluster-2 LOCAL adversary pass-1, F-C2-P1-001, EC-018).** Retroactive roll (Postcondition 7 catch point (i), over-cap `replace_all` write, `bytes_at_seal=49,500`, `shard_cap_bytes=49,152`) crashes after its own step (c) (canonical truncated to 0 bytes) but before its own step (d) (index publish) | `E-SHD-007`: on the next dispatch attempt, the gate finds `decision-log.0001.md` (49,500 bytes) on disk and absent from the index; reconciliation computes `bytes_at_seal=49,500 > shard_cap_bytes=49,152` (Invariant 7) and appends the recovered `[[shard]]` entry with `sealed_retroactively=true` — never a hardcoded `false` | error |

## Verification Properties

| VP-NNN | Property | Proof Method |
|--------|----------|-------------|
| VP-118 | Seal-then-block ordering invariant — the shard-index TOML always contains the seal entry for a given roll BEFORE (or atomically with) the corresponding `HookResult::Block` is observed by the caller | integration test (dispatcher harness: assert index file content immediately upon receiving the Block result) |
| VP-119 | No-over-cap invariant (no sealed shard file's byte size, sampled at any point after this BC's gate executes, ever exceeds its recorded `shard_cap_bytes` at seal time); Stable-current-filename invariant (the canonical filename is never itself renamed to a sealed name across any sequence of rolls; `stat(canonical_path)` always succeeds — CORRECTED, F-P2-003: verified against the copy-then-atomic-truncate mechanism, which structurally cannot vacate the canonical path, rather than the withdrawn rename-away mechanism) | proptest — two facets, arbitrary sequence of writes / roll sequence against a simulated artifact: every sealed shard's `bytes_at_seal <= shard_cap_bytes`; AND `stat(canonical_path)` always succeeds and is never the sealed inode from a prior roll |
| VP-120 | Retry-wording determinism (the block reason's retry instruction is the SAME fixed unified template regardless of the original tool name — CORRECTED, F-P2-002: no longer a per-tool-name choice between two divergent wordings); Fail-loud shard-seal-write-failure invariant (`E-SHD-001`, Postcondition 1 steps (a)-(b) failure, `HookResult::Error`, canonical file left untouched, per EC-003) | unit test — two facets: table-driven over both tool-name classes, asserting identical template with tool-specific guidance embedded within it; injected shard-seal-write-failure FS asserting `E-SHD-001` + pre-roll-state preservation |
| VP-138 | Truncate-after-seal self-heal invariant (`E-SHD-006`) — a crash between Postcondition 1 step (b) (sealed-shard publish) and step (c) (atomic-truncate) resolves, on the NEXT dispatch attempt, to a byte-identity check against the sealed shard followed by resume-from-step-(c)-only recovery (truncate + index publish only; the already-correct sealed shard is never rewritten), per EC-010 | integration test (fault-injection across two dispatches: simulate a crash between steps (b) and (c); assert post-recovery state is exactly one sealed shard, one index entry, and an empty canonical file) |
| VP-139 | Index-after-truncate self-heal invariant (`E-SHD-007`) — a crash between Postcondition 1 step (c) (atomic-truncate) and step (d) (index publish) resolves, on the NEXT dispatch attempt, to a filesystem scan for un-indexed sealed shards followed by an append-only reconciliation of the missing `[[shard]]` entry, per EC-011 and Postcondition 5's schema | integration test (fault-injection across two dispatches: simulate a crash between steps (c) and (d); assert the reconciled index gains exactly the missing entry, existing entries untouched, idempotent on repeat) |
| VP-NNN (pending) | **NEW (F2 spec-evolution, replace_all closure; EXTENDED cluster-2 LOCAL adversary pass-1, F-C2-P1-001/002).** Bounded post-write reconciliation for under-projected `replace_all: true` writes (Postcondition 7) — actual-size `stat()` check at catch point (i)/(ii), retroactive four-step roll reuse (`E-SHD-001`/`E-SHD-006`/`E-SHD-007` self-healing applies identically), the `sealed_retroactively` audit flag (Postcondition 5), and the bounded-window guarantee (over-cap observable for at most one subsequent matched dispatch, never indefinitely), per EC-014/EC-015/EC-016. **EXTENDED scope (F-C2-P1-002):** catch point (ii)'s per-tool cost split — `Edit`/`MultiEdit` reuse BC-1.18.005's existing `stat()`, `Write` performs a NEW dedicated `stat()` before applying — and the data-loss-closure property that a `Write` following a crash-orphaned, un-sealed, over-cap canonical file NEVER applies directly against it (EC-017). **EXTENDED scope (F-C2-P1-001):** the `sealed_retroactively` recovery-by-inference rule (Invariant 7) — `E-SHD-007` self-heal reconciliation of an orphaned sealed shard MUST set `sealed_retroactively = (bytes_at_seal > shard_cap_bytes)`, never a hardcoded `false` (EC-018) | integration test (fault-injection: simulate a `replace_all` write whose true occurrence-multiplied size exceeds cap while BC-1.18.005's single-occurrence estimate does not; assert catch point (i) fires and reconciles; separately, simulate catch point (i) crashing/never-running and assert catch point (ii)'s next-dispatch backstop reconciles instead — table-driven over BOTH a following `Edit`/`MultiEdit` and a following `Write`, asserting the `Write` case performs its own dedicated `stat()` call and never applies its `content` against an over-cap canonical file (EC-017); separately, fault-inject a crash between a retroactive roll's own step (c) and step (d), and assert the self-heal-recovered `[[shard]]` entry's `sealed_retroactively` field equals `bytes_at_seal > shard_cap_bytes` (EC-018)). **Allocation OWED to Phase F6 targeted-hardening (architect/formal-verifier), mirroring BC-1.18.005 EC-017's own VP-owed-to-F6 precedent (S2502-CLUSTER1-PASS5 STATE.md Drift Item) — product-owner does NOT self-allocate a VP number in this burst.** |

**Fix-burst note (fix-burst pass-3, F-P3-006):** VP-118's and VP-119's previously-separate rows are
each collapsed to ONE row (multi-facet convention). The prior separate VP-118 "partial-failure
self-healing invariant" row (spanning `E-SHD-001`/`E-SHD-006`/`E-SHD-007`) is REMOVED — it
duplicated and OVER-CLAIMED coverage that VP-INDEX v3.04 authoritatively assigns per-code:
`E-SHD-001`'s fail-loud leg is VP-120's (already in this table, "fail-loud shard-seal-write error
`E-SHD-001`" facet — see VP Anchors below), `E-SHD-006` is VP-138's, and `E-SHD-007` is VP-139's
(both already listed as separate rows in this table). No test coverage is lost; only the
mis-attributed BC-side claim under VP-118 is removed. VP-118's remaining single row (seal-then-block
ordering) is its genuine, non-overlapping scope.

## Related BCs

- BC-1.18.005 — owns the size-trigger and cap formula that determines when this BC's roll fires (depends on)
- BC-1.18.007 — retention/compaction policy consumes this BC's shard-index schema to identify archival candidates (composes with)
- BC-1.18.008 — the one-time backfill-split reuses this BC's exact seal+create+index-publish sequence, applied retroactively (composes with)
- BC-1.18.009 — mechanism B1 (BC-INDEX changelog rotation) is triggered by the SAME native gate this BC defines, with a different artifact-shape case (composes with)
- BC-1.18.010 — mechanism B2 (BC-INDEX per-subsystem sharding) is triggered by the SAME native gate, keyed by a manifest instead of the stable-filename trick (composes with)
- BC-1.18.012 — the governed one-time B1 changelog backfill migration reuses this BC's staging discipline pattern (temp-file-then-rename atomic-write) analogous to BC-1.18.008's relationship to this BC (related to)

## Architecture Anchors

- `crates/factory-dispatcher/src/shard_manager.rs` (new module) — publish-sealed-shard+atomic-truncate-canonical+index-publish sequence (CORRECTED, F-P2-003: copy-then-atomic-truncate, not rename-away), `HookResult::Block` construction with the unified retry-instruction text
- `crates/hook-sdk/src/result.rs` — `HookResult` enum (`Continue`/`Block { reason }`/`Error { message }`), the structural contract motivating block-and-retry over transparent redirection
- `crates/factory-dispatcher/src/indeterminate_marker.rs` — `write_indeterminate_marker`'s temp-file-then-rename atomic-write pattern, reused for shard-index publication
- `crates/last-amended-migrate/src/atomic_write.rs` — `write_atomic`, the alternative existing atomic-write primitive this BC's implementation may reuse instead of duplicating `indeterminate_marker.rs`'s
- `crates/factory-dispatcher/src/shard_manager.rs` (extends the existing module, F2 spec-evolution, replace_all closure) — Postcondition 7's post-write reconciliation: catch point (i), a native PostToolUse-side `stat()` check alongside the existing native PreToolUse handling BC-1.18.005 Precondition 1 already establishes; and catch point (ii), a leading `stat()`-comparison probe within the existing native PreToolUse handling path, before BC-1.18.005's own trigger evaluation. Both reuse Postcondition 1's existing seal+atomic-truncate+index-publish sequence verbatim — no new atomic-write primitive, no new `HookResult` variant, no new `hooks-registry.toml` entry (this remains native dispatcher code, not a WASM plugin, per the existing "why native, not WASM" rationale, BC-1.18.005 Postcondition 2)

## SDK Grounding Evidence

Literal stable-anchor greps substantiating this BC's external-artifact claims (POLICY 5;
no `grep -n` / no file:line citations per TD-VSDD-091):

```
$ grep -oE "^pub enum HookResult" crates/hook-sdk/src/result.rs
pub enum HookResult
```

```
$ grep -oE "^\s*(Continue|Block \{[^}]*\}|Error \{[^}]*\})" crates/hook-sdk/src/result.rs | sed -E 's/^\s+//' | sort -u
Block { reason: String }
Continue
Error { message: String }
```

Confirms the exact three-variant `HookResult` contract this BC's Precondition 3 and Invariant 1
depend on — no fourth "redirect" variant exists, grounding the "transparent redirection is
structurally impossible" claim in the Description.

```
$ grep -oE "^pub fn write_indeterminate_marker|^pub fn block_if_marker_check|^pub fn should_write_marker" crates/factory-dispatcher/src/indeterminate_marker.rs
pub fn block_if_marker_check
pub fn should_write_marker
pub fn write_indeterminate_marker
```

Confirms `write_indeterminate_marker`'s existence as the temp-file-then-rename atomic-write
precedent this BC's Postcondition 1 steps (b)-(d) and Architecture Anchors cite.

```
$ grep -oE "^pub fn write_atomic" crates/last-amended-migrate/src/atomic_write.rs
pub fn write_atomic
```

Confirms the alternative atomic-write primitive (`write_atomic`) this BC's Architecture Anchors
name as a reuse candidate.

## Story Anchor

S-25.02 — Artifact Sharding Layer 2: Size-Triggered Shard Rotation for Cycle Artifacts

## VP Anchors

- VP-118, VP-119, VP-120 — allocated by formal-verifier (S-25.02 F2 verification-property extension burst; VP-INDEX v3.02). VP-118 (integration; publish-sealed-shard→atomic-truncate-canonical→atomic-index-publish before Block + same-invocation atomicity + NEW partial-failure self-healing invariant per fix-burst pass-2), VP-119 (proptest; no-over-cap + stable-current-filename — re-verified against the corrected copy-then-atomic-truncate mechanism), VP-120 (unit-test; retry-wording determinism — re-verified as a single unified template, not a per-tool-name choice — + fail-loud shard-seal-write error E-SHD-001 + NEW E-SHD-006/E-SHD-007 partial-failure codes). Formal-verifier should review VP-118/119/120 bodies against this fix-burst's corrected mechanics (copy-then-truncate instead of rename-then-create; unified retry wording; staged 4-step sequence) — not yet actioned in this burst.
- VP-138, VP-139 — allocated by formal-verifier (S-25.02 F2 verification-property fix-burst pass-2; VP-INDEX v3.04, F-P2-004 partial-failure-code symmetry: every E-SHD code now has a VP leg). VP-138 (integration; Postcondition 1 step (c)/EC-010/Invariant 2/Invariant 3 — E-SHD-006 self-healing resume-from-truncate), VP-139 (integration; Postcondition 1 step (d)/EC-011/Postcondition 5 — E-SHD-007 self-healing index reconciliation). Back-references added S-25.02 F2 residual-cleanup micro-burst (formal-verifier's VP-138/VP-139 bodies already cited this BC in `source_bc`; this BC's own Verification Properties table and VP Anchors list did not yet cite them back — gap closed here, reference-only, no behavior change).
- VP-NNN (pending, F2 spec-evolution, replace_all closure; EXTENDED cluster-2 LOCAL adversary pass-1, F-C2-P1-001/002) — Postcondition 7's bounded post-write reconciliation for under-projected `replace_all: true` writes (catch point (i)/(ii) `stat()`-based reconciliation, retroactive four-step roll reuse, `sealed_retroactively` audit flag, bounded-window guarantee); EXTENDED to cover catch point (ii)'s per-tool `stat()` cost split and `Write`-path data-loss closure (EC-017, Invariant 7's sibling scope) and the `sealed_retroactively` recovery-by-inference rule for `E-SHD-007` self-heal (EC-018, Invariant 7). Allocation routed to architect/formal-verifier at Phase F6 targeted-hardening, mirroring BC-1.18.005 EC-017's own VP-owed-to-F6 precedent (S2502-CLUSTER1-PASS5 STATE.md Drift Item) — NOT self-allocated by product-owner in this burst.

## Traceability

| Field | Value |
|-------|-------|
| L2 Capability | CAP-043 |
| Capability Anchor Justification | CAP-043 ("Artifact Sharding Layer 2: Size-Triggered Shard Rotation for Cycle Append-Logs and BC-INDEX Structured-Catalog Sharding") per capabilities.md §CAP-043 — this BC specifies CAP-043's roll-before-write mechanics: "performs a roll-before-write (publish a sealed shard copy as a new file, then atomically replace the canonical file's content with empty, then atomically publish the updated shard index) and returns `HookResult::Block` with an explicit, actionable retry instruction (transparent write-redirection is not implementable under `HookResult`'s ... contract)." (CORRECTED, fix-burst pass-2, F-P2-003, from the withdrawn "seal the current shard by rename, create a fresh empty current file" wording). |
| L2 Domain Invariants | none (dispatcher runtime architectural invariant, not an L2 domain-spec DI-NNN) |
| Architecture Module | SS-01 (Hook Dispatcher Core — `shard_manager.rs` roll/block sequence) |
| ADR | ADR-051 §Decision 1 (block-and-retry mechanism, full algorithm, v1.2 per-tool `projected_size` correction; PreToolUse-only scope for the core roll — Postconditions 1-6 remain governed here); §Decision 3 (stable-current-filename addressing, v1.2 copy-then-atomic-truncate correction); §Decision 4 (shard-index schema, extended v1.4 with `sealed_retroactively`); §Decision 11 (staged partial-failure sequence + E-SHD-006/007, fix-burst addition); §Decision 12 (append-only-tail assumption + sealed-shard escape hatch, fix-burst addition); **§Decision 15 (NEW, ADR-051 v1.8→v1.9, architect addendum) — governs Postcondition 7's catch point (i): the PostToolUse-side native check leg, which §Decision 1 did NOT originally scope (§Decision 1 is PreToolUse-only and had no retroactive-roll semantics); §Decision 15 documents that this leg performs no `HookResult` signaling (a pure side-effect check per EC-016) and specifies its reuse of the retroactive four-step roll**; §Context (`HookResult`'s three-variant SDK constraint). Catch point (ii) (the next-dispatch backstop probe) remains governed by §Decision 1's existing PreToolUse scope, since it runs as a leading step within that same existing handling path. |
| Stories | S-25.02 |
| Cycle | v1.0-brownfield-backfill (F2 — product-owner spec-evolution burst) |
| Feature | E-25 — Validation Integrity and Large-Artifact Resilience |
| Deferred-Gap Closure | v1.4 CLOSES the `replace_all: true` occurrence-multiplicity gap BC-1.18.005 v1.11 Postcondition 3 explicitly DEFERRED to this BC's own cluster-2 F2 spec-evolution burst (S2502-CLUSTER1-PASS5 STATE.md Drift Item). |

## Changelog

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.5 | 2026-09-07 | product-owner | Fix-burst amendment resolving three MAJOR findings from S-25.02 cluster-2 LOCAL adversary pass-1 (F-C2-P1-001, F-C2-P1-002, F-C2-P1-004). **F-C2-P1-002 (catch point (ii) internal contradiction + real data-loss path):** catch point (ii)'s claim that it reuses "the same `stat()` BC-1.18.005 Postcondition 2 already performs" for EVERY subsequent `Edit`/`Write`/`MultiEdit` was TRUE for `Edit`/`MultiEdit` but WRONG for `Write` (whose Postcondition 3 trigger, `projected_size = len(content)`, performs no `stat()` at all) — left uncorrected, this allowed a subsequent under-cap `Write` to apply directly against a crash-orphaned, un-sealed, over-cap canonical file, destroying that history with no roll ever having started for self-heal to catch. ADJUDICATED under the production-grade default (no version of this gate may lose history): catch point (ii) now performs its OWN dedicated, bounded `stat()` of the canonical file specifically for the `Write` arm (BC-1.18.005's stat-free `Write` trigger formula is UNCHANGED — this is an additional backstop probe, not a change to the trigger); `Edit`/`MultiEdit` remain cost-free (reuse the existing stat). Corrected the contradictory wording in catch point (ii) and in Postcondition 7's "Scope" paragraph (which had over-generalized catch point (i)'s `replace_all`-only scope to imply `Write` pays zero cost anywhere in Postcondition 7). ADDED EC-017 and a matching Canonical Test Vector for the `Write`-next-dispatch backstop case. **F-C2-P1-001 (`sealed_retroactively` mislabeled on a crashed retroactive roll):** if a RETROACTIVE roll (catch point (i)/(ii)) crashes between its own step (c) and step (d), `E-SHD-007`'s self-heal reconciliation (scanning the filesystem for un-indexed sealed shards with no other context) would otherwise hardcode the recovered entry's `sealed_retroactively` to `false`, mislabeling a genuinely over-cap shard as cap-guaranteed. ADDED NEW Invariant 7 specifying the deterministic inference rule (`bytes_at_seal > shard_cap_bytes ⇒ sealed_retroactively = true`, since a prospective roll can never seal over-cap content) and REQUIRING `E-SHD-007` self-heal reconciliation to apply it. Cross-referenced from Postcondition 1's `E-SHD-007` recovery text and Postcondition 5's `sealed_retroactively` field description. ADDED EC-018 and a matching Canonical Test Vector. **F-C2-P1-004 (stale Canonical Test Vector, POLICY 12):** the first CTV row's numeric example (`Write`, current shard 45,000 bytes, `content` length 5,000 bytes, cap 49,152 → `Block`) was a stale carry-over from the WITHDRAWN `current + payload` formula and was inconsistent with the corrected `projected_size = len(content)` Write formula (5,000 <= 49,152 would `Continue`, not `Block`). REPLACED with a self-consistent example (`content` length 50,000 bytes, cap 49,152) that actually triggers the roll under the current formula, clarifying that the sealed shard's size is always the pre-roll canonical content (45,000 bytes), never the blocked `Write`'s own `content`. Extended the pending VP-NNN row and VP Anchors entry to cover the extended scope (EC-017/EC-018/Invariant 7); allocation remains routed to Phase F6 (not self-allocated). No change to Postconditions 1-6, 3-6, or any other Invariant, Edge Case, or Canonical Test Vector beyond those named above. `status` remains `draft`. |
| 1.4 | 2026-09-07 | product-owner | F2 spec-evolution burst (S-25.02 cluster-2 pre-TDD): CLOSES BC-1.18.005 v1.11 Postcondition 3's deferred `replace_all: true` occurrence-multiplicity gap (S2502-CLUSTER1-PASS5 STATE.md Drift Item), adopting Option (b) of BC-1.18.005's own pre-authorized closure fork — BC-1.18.005's PreToolUse trigger formula is left UNCHANGED (Option (a), amending BC-1.18.005's already-ACTIVE/shipped contract, was NOT taken). ADDED Precondition 4 (closure-scope statement, `Edit`/`MultiEdit` `replace_all: true` calls only). ADDED Postcondition 7 (bounded post-write reconciliation): two redundant catch points — (i) an immediate post-write `stat()`-based check reusing Postcondition 1's exact four-step roll sequence retroactively, and (ii) a next-dispatch leading-probe backstop covering an (i)-crash scenario; a precise, testable bounded-window postcondition (canonical file over-cap observable for at most one subsequent matched dispatch, never indefinitely); and a documented, narrowly-scoped exception allowing a RETROACTIVELY-sealed shard's `bytes_at_seal` to exceed `shard_cap_bytes`, while the CANONICAL file's own zero-bytes-after-roll guarantee remains unconditional (codified in NEW Invariant 6). EXTENDED Postcondition 5's shard-index schema with an optional `sealed_retroactively` boolean field (default `false`, backward compatible) as the audit trail distinguishing the two shard-cap guarantee regimes. ADDED EC-014 (under-projected `replace_all` triggers catch point (i)), EC-015 (catch point (i) failure triggers catch point (ii) backstop), EC-016 (no agent-facing block/retry signal for a retroactive roll — unchanged from existing EC-002 scope) and three matching Canonical Test Vectors. ADDED one new Verification Property row, VP-NNN (pending) — allocation explicitly routed to architect/formal-verifier at Phase F6 targeted-hardening (NOT self-allocated), mirroring BC-1.18.005 EC-017's own VP-owed-to-F6 precedent. Architecture Anchors extended with the two new catch-point call sites (both native dispatcher code reusing Postcondition 1's existing mechanism; no new `HookResult` variant, no new atomic-write primitive, no new `hooks-registry.toml` entry). Traceability gained a "Deferred-Gap Closure" row citing the closed drift item. `status` remains `draft` (cluster-2 not yet shipped). **SAME-BURST CORRECTION (architect implementability-gate finding, post-initial-v1.4-draft):** this entry originally claimed "no new ADR decision required — contained within ADR-051's existing native-check pattern" for Postcondition 7. That OVERCLAIMED: ADR-051 §Decision 1 scoped the native check to PreToolUse ONLY, with no PostToolUse leg and no retroactive-roll semantics — catch point (i) (the PostToolUse-side check) was NOT already covered. The architect added **ADR-051 §Decision 15** (ADR-051 v1.8→v1.9) as a new addendum documenting the PostToolUse-side native-check leg, its pure-side-effect (no `HookResult` signaling, per EC-016) contract, and its reuse of the retroactive four-step roll. The Traceability ADR row is corrected accordingly: Postcondition 7 catch point (i) is now cited to §Decision 15; the core roll mechanism (Postconditions 1-6) and catch point (ii) remain under §Decision 1/§Decision 11. No postcondition, invariant, edge-case, or mechanism text changed — citation-accuracy correction only; BC stays at v1.4 (same-burst fix, not a new increment). |
| 1.3 | 2026-09-05 | product-owner | Fix-burst amendment (adversary pass-3 finding F-P3-006 LOW): collapsed the §Verification Properties table's separate VP-118 (×2) and VP-119 (×2) rows into one row each (multi-facet convention). CRITICALLY, REMOVED the second VP-118 "partial-failure self-healing invariant" row entirely — it duplicated and OVER-CLAIMED coverage that VP-INDEX v3.04 authoritatively assigns per-code to VP-120 (`E-SHD-001`), VP-138 (`E-SHD-006`), and VP-139 (`E-SHD-007`) respectively; folded the `E-SHD-001` fail-loud facet explicitly into VP-120's row to match VP Anchors' existing description. No test coverage lost, no postcondition/invariant/edge-case content change — table presentation and mis-attribution correction only. |
| 1.2 | 2026-09-05 | product-owner | Fix-burst amendment (adversary pass-2 findings F-P2-003 HIGH + F-P2-002 HIGH + F-P2-004 MEDIUM + F-P2-005 MEDIUM, ADR-051 v1.2 Decisions 1/3/11/12): (1) REWROTE Postcondition 1(a)/Invariant 2/Invariant 3 from the WITHDRAWN rename-away seal mechanism (which opened a real ENOENT window on the canonical path) to the CORRECTED copy-then-atomic-truncate-in-place mechanism — publish the sealed shard as a new file, then atomically replace the canonical file's content with empty via `write_atomic`'s temp-file-then-rename primitive; the canonical path is never absent. (2) REWROTE Postcondition 2/Invariant 4 to a SINGLE UNIFIED retry-instruction template, withdrawing the divergent `Write`-"simply retry unchanged" wording (unsound: could permanently deadlock a blocked `Write` whose stale pre-roll `content` remains over cap under the corrected `projected_size = len(content)` formula). (3) ADDED the staged 4-step per-write roll sequence (read/publish-sealed/atomic-truncate/publish-index) with two NEW partial-failure error codes `E-SHD-006` (seal published, canonical not yet truncated — self-healing resume-from-truncate) and `E-SHD-007` (canonical truncated, index not yet updated — self-healing index reconciliation), plus new EC-010/EC-011 and a new VP-118 fault-injection property. (4) ADDED Invariant 5 (append-only-tail assumption, explicit) and two new edge cases EC-012 (sealed-shard direct-edit escape hatch for already-relocated content) and EC-013 (still-mutable tail edit, unaffected by Invariant 5). Updated Canonical Test Vectors, Architecture Anchors, SDK Grounding cross-references, VP Anchors, and Traceability's Capability Anchor Justification quote and ADR citation accordingly. Added BC-1.18.012 to Related BCs. |
| 1.1 | 2026-09-05 | product-owner | Fix-burst amendment (F-S2502-F2-007, POLICY 5): added `## SDK Grounding Evidence` section with literal stable-anchor grep output for `HookResult`'s three-variant enum, `write_indeterminate_marker`/`block_if_marker_check`, and `write_atomic`. No postcondition/invariant/VP content change — this BC's contract was confirmed unaffected by the sibling BC-1.18.009 BLOCKER fix (architect: "No change required," F2 architecture-delta §4a). |
| 1.0 | 2026-09-05 | product-owner | Initial creation. F2 spec-evolution burst, S-25.02 activation. Encodes the CORRECTED block-and-retry roll semantics (per ADR-051's finding that `HookResult`'s Continue/Block/Error contract forbids transparent write-redirection) rather than a transparent-redirect fiction; shard-index schema; stable-current-filename addressing as a structural consequence of the seal mechanism. CAP-043 capability anchor. ADR-051 §D1/§D3/§D4 citations. |
