---
document_type: research-brief
topic: crash-safe atomic multi-file publication + single-writer exclusion + executable-integrity
grounds: ADR-052 v1.2 -> v1.3 redesign
producer: research-agent
date: 2026-09-13
sources:
  - /tmp/codex-adr052-v12/codex-review.json (3rd cross-vendor adversarial review, 11 findings)
  - .factory/specs/architecture/decisions/ADR-052-native-migration-cli-bash-tool-allowlist-sanctioned-execution-path.md (v1.2)
scope: research output only (no ADR/BC/STATE/index edits; no git operations)
---

# Research Brief — ADR-052 v1.3: Crash-Safe Atomic Publication, Single-Writer Exclusion, Executable Integrity

## Purpose and framing

The 3rd cross-vendor (Codex) adversarial review returned NOT RATIFIABLE with 11 findings, all
rooted in one architectural choice: ADR-052 v1.2 publishes a multi-file migration by renaming each
target file **individually in place** (`rename(staging_path, target_path)` per target, §Decision 7c),
tracking per-target progress in a `completed_renames` list, and using a `COMMITTED` phase-marker as
a *logical* commit-pointer that the reader must consult. The review's core thesis is that a
per-file in-place rename model **cannot** provide all-or-nothing multi-file visibility, cannot
cleanly recover an interrupted rename sequence, and entangles OS-process lock ownership with
persistent transaction intent.

This brief maps each finding cluster to the established systems-engineering pattern that resolves
it, cites authoritative sources, and gives a short applicability note for the `.factory/`
git-worktree context on macOS (darwin-arm64) and Linux. It ends with a concrete "Recommended v1.3
direction" the architect can implement against.

**Finding index (from codex-review.json, in order):**
F1 preserve immutable legacy read generation; F2 recover renames completed before journal update;
F3 drain admitted writers before reading inputs; F4 serialize stale-lock reclamation + incomplete
lock creation; F5 exclusive takeover of dead owner's PREPARED transaction; F6 retain recovery
state when authorization expires; F7 native gate Bash mutation-target classification; F8 census /
terminal-rerun reachability through guard stack; F9 retain unambiguous terminal publication record
after cleanup; F10 authorize all migration output paths consistently; F11 bind executable
verification to actually-executed artifact.

**Cross-cutting root cause:** F1, F2, F5, F9 all dissolve if publication becomes a single atomic
pointer swap over an immutable generation, rather than N in-place renames. That is the central
recommendation.

---

## RQ1 — Atomic multi-file publication (resolves F1, F2, F9; supports F5, F10)

### The finding, precisely

- **F1:** During the window between renaming BC-INDEX.md and writing COMMITTED, a reader that finds
  "COMMITTED absent" is told to "read from legacy BC-INDEX.md" — but BC-INDEX.md has *already* been
  replaced with its shard-redirect body. The reader gets the new body while being promised the old.
  This window persists indefinitely after a crash.
- **F2:** A crash after a `rename()` but before the `completed_renames` journal append leaves a
  replaced target, an absent staging file, and no completion record. The specified retry cannot
  recover it (staging is gone; the journal says "untouched").
- **F9:** After CLEANED archives COMMITTED, a fresh reader/rerun cannot distinguish "steady state,
  migration long done" from "never ran," because both present as "COMMITTED absent."

All three are symptoms of publishing by mutating live canonical paths one at a time. There is no
single point at which the whole set flips.

### Recommended pattern: single atomic pointer swap over an immutable generation

The robust POSIX pattern is: **build a complete, immutable generation off to the side, durably sync
it, then publish it with exactly one atomic `rename()` of a pointer/manifest file (or one atomic
symlink swap). Readers resolve that pointer once and pin the resulting generation for the entire
operation.** Renaming each payload file separately is not a multi-file transaction and lets readers
observe mixed generations. [man7 rename(2); OSDI'14 Pillai et al.]

Key mechanics, all corroborated by ≥2 sources:

1. **Layout.** Store each version as an immutable set, e.g. `generations/G<id>/{BC-INDEX.md, shards/...}`.
   Publish by writing the new generation id to `CURRENT.tmp`, syncing, then
   `rename("CURRENT.tmp","CURRENT")` — or by creating `current.tmp -> generations/G<id>` and renaming
   that symlink over `current`. The single rename is the visibility/commit point: a reader opening the
   pointer sees either wholly-old or wholly-new. Nix profiles are established prior art (build new
   generation, then atomically flip the profile symlink). [nix.dev profiles; deployer.org atomic-symlinks]
2. **Why per-file rename is insufficient.** Publishing a, then b, then c with three renames lets a
   reader between operations observe new-a with old-b/old-c. POSIX provides no general "atomically
   rename this *set* of unrelated names" operation. Linux `RENAME_EXCHANGE` swaps exactly two existing
   pathnames, not an arbitrary set. Git's own docs explicitly warn that a concurrent reader can observe
   only a *subset* of a multi-ref update — i.e., multiple pathname replacements do **not** compose into
   one reader-visible transaction. [man7 rename(2); git-scm update-ref]
3. **Reader pinning.** Two established techniques: (a) *manifest* — open `CURRENT` once, read+validate
   one generation id, then access every file under `generations/<that-id>/…`, never re-reading CURRENT
   mid-operation; (b) *directory fd* — resolve `current` once, retain the directory fd, and open members
   with `openat(genfd, "…")`. A directory fd is a stable reference even if the directory is later
   renamed, and `openat` avoids re-resolving the changing prefix. Merely using paths like `current/a`
   then `current/b` is unsafe: the symlink can flip between the two lookups. [man7 openat(2)]
4. **Copy-on-write generation dirs + safe GC.** Build `generations/.G<id>.tmp`, write+verify all
   contents without touching the live generation, sync, rename to immutable `generations/G<id>`, then do
   the single pointer swap. Old-generation cleanup must not delete a generation a reader may still hold:
   use a shared-reader/exclusive-GC lock, refcounts/leases, epoch/RCU-style grace, or (simplest) pre-open
   every file a reader needs so open-but-unlinked inodes survive until close. LMDB uses the analogous
   proven design — readers record transaction ids; pages reachable by the oldest reader cannot be
   reclaimed. [lmdb.tech SDC'15; man7 unlink(2)]

### Prior-art commit points (all verified)

| System | Atomic commit point |
|---|---|
| SQLite (rollback journal) | Deleting/invalidating the rollback journal; for multi-file ATTACH txns, deleting a **super-journal** that names every participant is the single multi-file commit point. SQLite syncs the super-journal's *directory* on Unix. [sqlite.org/atomiccommit] |
| LMDB | Copy-on-write B+tree; commit = updating one of two alternating **meta pages**; readers pick the valid meta page with the greatest txn id. [lmdb.tech] |
| Git | Ref/HEAD update changes one OID pointer atomically; index uses `index.lock` create-exclusive → write → rename to `index`. Multi-ref is *not* reader-atomic (per Git docs). [git-scm] |
| systemd / dpkg | Write-temp-then-rename for **one** file (dpkg: `pathname.dpkg-new` → rename; default safe-I/O syncs before rename). Not whole-package atomic visibility. [manpages.debian dpkg; freedesktop systemd] |
| maildir | Write fully under `tmp/`, then move to `new/`; readers ignore `tmp/`. Atomic publication of one message, not a whole-mailbox snapshot; requires single device. [cr.yp.to/proto/maildir] |

### POSIX rename(2) — what is and is NOT guaranteed (verified man7 + LWN)

- **Guaranteed:** if `newpath` exists it is atomically replaced; an observer looking up `newpath`
  never sees it absent and gets either old or replacement object. Existing fds stay attached to their
  original inode. On failure an instance of `newpath` remains. [man7 rename(2)]
- **Same-filesystem only:** cross-filesystem rename fails `EXDEV` (can occur even between two mounts of
  the same fs). Keep `CURRENT.tmp` and `CURRENT` in the same directory. [man7 rename(2)]
- **NOT guaranteed:** atomic publication of several independent names; that the replacement's *data* was
  written; ordering of data before the directory-metadata change; survival across power loss. "Namespace
  atomicity is not durability." [LWN 323169; OSDI'14 Pillai et al., which found persistence properties
  varied across 6 Linux filesystems and 60 crash-vulnerabilities in 11 apps]

### Directory fsync for durability (Linux verified; macOS flagged — see RQ-macOS)

On Linux ext4/xfs the canonical durable-rename sequence is: `write tmp → fsync(tmp) → rename(tmp,dst)
→ fsync(parent_dir)`. `fsync(file)` persists the file's data+inode but **not** necessarily the
containing directory entry, so an explicit directory fsync is required. LWN documents exactly this
`open/write/fsync/close/rename/fsync(dir)` sequence. [man7 fsync(2); LWN 457667/458176]

### Applicability to `.factory/` git-worktree context

- The `.factory/` tree is a git worktree of the orphan `factory-artifacts` branch. A generation
  directory (`generations/G<id>/`) and its pointer must live on the **same filesystem** — they do, since
  they'd both sit under `.factory/`. `rename()` of the pointer is same-directory, so no `EXDEV` risk.
- A **symlink pointer** is awkward here: git tracks symlinks, and the shard files are the artifacts that
  must be committed and diffable. Prefer a **manifest-file pointer** (a small committed file naming the
  current generation / or the canonical shard set) over a directory symlink. This keeps the published
  content as ordinary tracked files.
- Caveat: the "immutable generation directory + pointer swap" model changes the *on-disk shape* of the
  BC-INDEX shard tree. If the architect wants shards to remain at fixed canonical paths
  (`shards/BC-INDEX-SS-NN.md`) for downstream tooling, the lighter-weight adaptation is: keep a single
  durable **terminal-state manifest** (see F9 below) as the authoritative "which layout is live" record,
  and treat the multi-file replacement as staged-then-swapped via one manifest write rather than N
  reader-visible in-place renames. Either way, the fix for F1/F9 is a **single durable record that names
  the live generation**, resolved once per reader operation — replacing the "COMMITTED absent ⇒ read
  legacy" heuristic that is ambiguous both mid-migration and in steady state.

---

## RQ2 — Write-ahead journal / intent-log recovery (resolves F2; supports F6)

### Recommended pattern: durable redo/intent log with idempotent forward recovery

The established pattern is the filesystem analogue of the database **WAL rule** (recovery info must
reach stable storage before the update it protects; PostgreSQL: "data files only after WAL records
describing the changes have been flushed"). [postgresql.org/docs wal-intro] Concretely:

1. Create staging file (same fs, ideally same directory as target).
2. Write all bytes + metadata to staging.
3. Compute + verify expected post-op hash `H`.
4. `fsync(staging_fd)` (+ `fsync(staging_parent_dir)` if newly created). — **durable barrier**
5. Append an **INTENT** record per file: txn id, staging path, target path, expected hash `H`,
   object type/size, **and the expected target pre-state** (missing | old-hash | generation).
6. `fsync(journal_fd)`. — **WAL boundary: after this, every destructive action is recoverable.**
7. `rename(staging, target)` (same fs).
8. Sync target parent dir (+ staging parent dir for cross-dir rename).
9. Append `DONE(txn, file)`; `fsync(journal_fd)`.
10. Cleanup staging / compact journal (itself an idempotent, separately-durable phase).

**Records must be framed + checksummed; a torn final record must never be interpreted as an INTENT
or DONE.** Ext4 fast-commit and XFS both do result-oriented, idempotent recovery: XFS records logged
*intents* and later *done* items; on recovery an intent without its done item is reconstructed and
finished. [kernel.org ext4/journal.rst; xfs-online-fsck-design]

### The critical F2 resolution: accept a matching destination hash as completion

F2's exact gap — "rename done, DONE record not yet written, staging gone" — is resolved by the
recovery rule **"enforce and verify the intended resulting state; do not blindly re-execute the
procedure."** Recovery decision table (synthesized from ext4/XFS/ARIES principles; the specific hash
decision matrix is application protocol, not a POSIX guarantee — flagged):

| Destination `D` | Staging `S` | Safe decision | Why |
|---|---|---|---|
| `D` == `H` (+ metadata) | missing | **Treat complete; persist DONE** | canonical "rename done, crashed before DONE" |
| `D` == `H` | `S` == `H` | **Treat complete; DONE; quarantine/remove staging** | postcondition already true; re-rename adds risk |
| `D` != `H`, `D` matches logged precondition `P` | `S` == `H` | **Redo rename, sync dir, verify, DONE** | intent durable, rename not yet done |
| `D` missing, `P` says missing | `S` == `H` | **Redo rename** | clean initial-publication case |
| `D` unexpected bytes, no `P` logged | `S` == `H` | **FAIL CLOSED** unless single-writer exclusion is independently guaranteed | cannot distinguish old target from a legit post-crash update |
| `D` != `H` | `S` missing | **FAIL CLOSED** | neither result nor recovery copy exists |
| durable DONE exists but `D` != `H` | any | **FAIL CLOSED; don't trust DONE blindly** | later modification / ordering bug / broken durability assumption |
| torn/invalid journal frame | any | **FAIL CLOSED** | no trustworthy authority for a destructive decision |

Accepting `hash(D)==H` as completion (a) closes the rename/ack gap, (b) makes replay idempotent
(recovery can itself crash and re-run), (c) removes dependence on staging survival, (d) separates
truth (the destination content) from bookkeeping (DONE). This is safe **only if** the journal record
is valid+durable, the hash is collision-resistant over the full byte stream+length, all
semantically-relevant metadata is separately checked, semantics are "ensure this path has this
state" (not "prove this inode passed through rename exactly once"), concurrent writers are excluded,
and recovery hashes a safely-*opened* regular file (not a re-resolved pathname). A content hash does
**not** prove provenance/inode identity — if those matter, add a signed manifest / generation number.

### Durable ordering + fault injection

Ordering that makes recovery sound: staging data durable **before** the journal points at it;
journal durable **before** rename; parent dir durable **before** DONE. Do not rely on mount modes
(ext4 `data=ordered`/`data=journal`, xfs log forces) as a substitute for these application-level
barriers — they are a lower-level safety net, not a replacement. [kernel.org admin-guide/ext4;
xfs-delayed-logging-design]

Validate with fault injection between **every** rename/fsync/journal-write, and repeat the campaign
while running recovery itself (a second recovery pass must converge to the same state). Tools: ALICE
(system-call-trace crash-state exploration), CrashMonkey/ACE (block-I/O crash images), dm-log-writes
(record every write + FUA/flush marks, replay to marks). [OSDI'14 Pillai; Microsoft Research
CrashMonkey; kernel.org dm-log-writes]

### Applicability to `.factory/`

The migration binary already plans a PREPARED marker with `source_sha256`, `completed_renames`, and
`staging_paths` (§7b). Upgrading that to a **framed, checksummed intent log that records per-target
expected post-content hash + expected pre-state, with the accept-matching-destination-as-complete
recovery rule**, directly closes F2 without inventing new infrastructure. The single most important
change: **record the expected post-hash per target BEFORE the first rename**, and make recovery
verify destination hashes rather than relying on staging files that no longer exist.

---

## RQ3 — Single-writer exclusion / advisory locking (resolves F3, F4, F5)

### Recommended pattern: advisory lock on a stable, never-unlinked inode

**For cooperative single-writer exclusion on local Linux/macOS, use an exclusive `flock()` on a
dedicated, stable, never-unlinked lock inode, holding its fd for the entire critical section.** On
Linux, a whole-file OFD lock (`fcntl F_OFD_SETLK`, range 0..EOF) is equally robust and adds byte-range
control. **Avoid traditional `fcntl F_SETLK`** when unrelated code might open/close the same inode.
[man7 flock(2), fcntl(2)]

| Mechanism | Ownership | Key hazard | Verdict |
|---|---|---|---|
| `flock(LOCK_EX)` | open file description (OFD) | dup/fork share it; separate `open()` calls contend; survives `execve` unless `O_CLOEXEC` | **Best cross-platform default.** Immune to the POSIX any-close defect. Apple documents flock/fcntl/lockf as mutually compatible on macOS. [Apple flock(2)] |
| `fcntl F_SETLK` (traditional) | process + inode | **closing ANY fd to that inode in the process releases ALL the process's locks on it** — even an unrelated library's open/close | **Least safe** for app-wide exclusion. [man7 fcntl(2)] |
| `fcntl F_OFD_SETLK` | OFD (Linux 3.15+) | released only when last fd to the OFD closes; no deadlock detection | **Best Linux choice with byte ranges;** availability on old macOS / network volumes uncertain — flagged. [man7 fcntl(2); Apple archived fcntl(2) does not document OFD] |

All three are **advisory** — every writer must obey the same protocol; they do not stop a buggy or
privileged writer. [man7 flock(2)]

### Why the v1.2 `O_CREAT|O_EXCL` lockfile has an inherent race (F4)

"Same file" means same **inode**, not same pathname. `unlink()` removes the name while open fds keep
referring to the old inode — which is exactly what enables split ownership. The concrete failing
interleaving the review describes:

1. A and B both decide inode `I` is stale.
2. A `unlink(I)`.
3. A creates + owns new inode `A` via `O_CREAT|O_EXCL`.
4. B executes its already-decided `unlink(path)` — **now removing A's live inode.**
5. B creates inode `B`. A and B both believe they hold the lock, on different inodes.

`O_CREAT|O_EXCL` makes one create-if-absent attempt fail `EEXIST`; it does **not** make stale-file
reclamation safe. With a **stable flock/OFD inode there is normally nothing to reclaim**: when the
last fd closes (including on process death/crash), the kernel releases the advisory lock and the next
contender acquires it atomically. PID metadata becomes *informational, not authoritative.*
[man7 open(2), unlink(2), flock(2)]

### Race-free stale-owner handling (F4)

`kill(pid,0)` only checks existence/permission (`ESRCH` = no such PID, `EPERM` = exists but
unsignalable) and **cannot** prove the numeric PID is the same process that wrote the metadata (PID
reuse). Storing `{pid, starttime (proc/PID/stat field 22), boot-id}` is stronger than PID alone; a
pidfd is a stable reference *once obtained* but doesn't retroactively prove identity. Crucially, the
sequence **read-PID → decide-dead → unlink → recreate is TOCTOU-racy even with perfect liveness
detection** and must not be used for correctness. The advisory-lock-on-stable-inode approach sidesteps
this entirely. [man7 kill(2), proc_pid_stat(5), pidfd_open(2)]

### Empty / corrupt lock metadata (F4)

`O_CREAT|O_EXCL` publishes the directory entry **before** the JSON body is written; a crash (or a
short `write()`) leaves a zero-length/truncated/malformed file — a **normal crash state, not proof of
no owner.** Fail-closed rule: **never classify malformed metadata as "dead, therefore unlink."**
First try to acquire the advisory lock. If acquisition fails → report *busy / owner-metadata-
unavailable*. If it succeeds → no live cooperating owner holds it, so the new owner may truncate and
rewrite diagnostic metadata under the held lock (loop on partial writes, validate a versioned schema +
txn UUID, fsync if durability matters). [man7 write(2)]

### Separate ephemeral process ownership from persistent maintenance intent (F5)

Keep **two distinct records**: (a) the OS advisory lock = "this running OFD/process is the sole
executor now"; (b) a durable transaction record = "maintenance transaction T is incomplete and must be
completed or rolled back." The durable record holds a txn UUID, state/phase, input generation, a
fencing/claim generation, timestamps, recovery info, and results — **not merely a PID.** On restart, a
recovery process acquires the same advisory lock, atomically claims the unfinished record via
version/CAS or by bumping its fencing generation, validates artifacts, resumes or rolls back, and only
then marks the record terminal. **Releasing/losing the OS lock must NOT delete the durable intent; a
stale intent does NOT by itself grant execution rights** (the recovery process must first obtain the
ephemeral lock). This is exactly the F5 fix: ordinary writers stay blocked for *every* unresolved
PREPARED transaction regardless of PID liveness, while a distinct recovery-owner acquisition updates
ownership without clearing intent. [postgresql WAL; Oracle in-doubt txn recovery — see RQ5]

### Drain admitted writers before snapshotting (F3)

A correct barrier distinguishes **admission** from **completion.** The v1.2 "TOCTOU fingerprint check
once before first rename" only detects writes that *completed* in a window; it does not wait for a
writer *admitted before the lock existed* to finish. Correct protocol:

1. Become sole maintenance coordinator (acquire the exclusive lock).
2. Atomically flip the admission gate `OPEN → DRAINING` (no new reservations).
3. Wait until the set/count of already-admitted reservations reaches zero (quiescence).
4. Snapshot the now-quiescent inputs.
5. Flip gate back to `OPEN` and wake waiters.

Each normal writer takes a **reservation before reading snapshot-sensitive inputs** and holds it until
its writes commit. Within one process, protect `{gate_state, active_count}` with one mutex + condvar
(`pthread_cond_wait` atomically releases the mutex while waiting, avoiding lost wakeups). Across
processes, put the gate/reservations in a transactional store or use a stable barrier inode (writers
hold a *shared* reservation for the whole op; maintenance closes admissions then takes *exclusive*).
**Do not rely on upgrading a shared flock to exclusive** — the conversion releases the old lock before
acquiring the new and is not atomic. For crashable writers, reservations must be renewable leases with
expiry + fencing. [man7 flock(2); Apple pthread_cond_wait(3)]

### Applicability to `.factory/`

- The dispatcher's native admission gate (§Decision 5a) is the natural place to enforce the DRAINING
  gate for **all** mutation tools (Edit/Write/MultiEdit/Bash) — which the ADR already routes there.
  The missing piece is a **writer-reservation lifetime** spanning PreToolUse → tool completion so the
  coordinator can *wait for* in-flight admitted writers, not merely detect completed ones. The review's
  F3 recommendation ("writer reservations held through actual mutation completion") maps to a
  PreToolUse-acquire / PostToolUse-release reservation counter in dispatcher state.
- The lock inode should be a **stable, pre-created, never-unlinked** file (e.g.
  `.factory/migration-state/exclusive.lock` created once and locked via flock, its JSON body treated as
  diagnostic-only). This is a direct replacement for the v1.2 `O_CREAT|O_EXCL` + PID-liveness +
  unlink-reclaim design that F4 shows is racy.
- Caveat: the dispatcher and agent tool calls are separate OS processes; flock across them works on
  local APFS/ext4 but the reservation *gate state* is dispatcher-internal. The architect must decide
  whether the gate lives in dispatcher memory (single dispatcher process) or a durable file (multiple).

---

## RQ4 — Executable verify-to-execute TOCTOU (resolves F11)

### The finding

A guard hashes `target/release/factory-dispatcher` at a path and compares to a stored digest
(§5c/§4), then a shell later executes that path **by name** (§Decision 1/3). Between check and exec,
the file can be rebuilt/replaced/symlink-swapped, so different bytes run. This is textbook CWE-367
TOCTOU (MITRE: check a resource whose state can change before use). [cwe.mitre.org/367]

### Recommended pattern: verify and execute the SAME opened fd, never the pathname

> **Open once → hash through that open fd → compare → execute through the same fd. Never return to
> the pathname.**

This defeats rename/rebuild/symlink-swap on the pathname. It does **not** by itself stop another
writable fd or shared mapping from modifying the *same inode* after hashing — that requires an
immutability mechanism as well. [man7 fexecve(3) explicitly states this limitation]

| Mechanism | Platform | Notes |
|---|---|---|
| `fexecve(fd, argv, envp)` | Linux (glibc), FreeBSD, POSIX.1-2008 | Executes program named by fd (fd may be `O_RDONLY` or `O_PATH`). glibc ≤2.26 used `/proc/self/fd`; ≥2.27 uses `execveat()`; without either can fail `ENOSYS`. Docs name checksum-before-execution as the *intended* use. [man7 fexecve(3)] |
| `execveat(fd,"",argv,envp,AT_EMPTY_PATH)` | Linux 3.19+ (glibc 2.34 wrapper) | Executes file referred to by fd instead of re-resolving pathname. [man7 execveat(2)] |
| `O_PATH` + hash caveat | Linux | `O_PATH` fds can't be read/mmap'd, so you can't hash through them; `fstat` is NOT a content hash. Open `O_RDONLY` to hash, then exec the same fd. [man7 open(2)] |
| memfd + `F_SEAL_WRITE/GROW/SHRINK/SEAL` (+ `F_SEAL_EXEC` on 6.3+) | Linux | Copy candidate into executable memfd, hash, seal, then fexecve. Gives a clean "bytes hashed == bytes executed" invariant. Linux-only. [man7 memfd_create(2)] |
| IMA/EVM `appraise func=BPRM_CHECK ... appraise_type=imasig` (enforce mode) | Linux | Kernel-level verify-at-exec on the file actually being loaded. Measurement alone only records hashes — must be appraisal in enforce mode. [kernel.org ima_policy] |

### macOS reality (FLAGGED — important for darwin-arm64)

- **Treat `fexecve` as UNAVAILABLE on macOS.** Apple's documented exec interfaces are all
  pathname-based (`execl/execle/execlp/execv/execvp/execvP/execve`); Apple's pages document neither
  `fexecve` nor `execveat`. FreeBSD *does* provide `fexecve`, so "BSD-derived" does not imply Darwin
  support. **AUTHORITATIVE-DOC FLAG:** I found no Apple document that *explicitly states* "macOS does
  not implement fexecve" — the strongest evidence is Apple's API/man-page set exposing only
  pathname-based exec; non-Apple SDK/source reports corroborate the missing symbol. Do not design macOS
  code assuming fexecve/execveat exist. [Apple execve(2), exec(3); FreeBSD fexecve(3)]
- **macOS fallback = protected staging-and-launch:** open+hash the source fd; copy into a
  root/trusted-owned staging dir under a fresh content-addressed name; reopen+rehash; set restrictive
  ownership + immutable flag; ensure the staging dir is not writable by the build/attacker user; execute
  the staged pathname directly from a trusted launcher (no intervening shell). This is namespace
  protection, not an fd-level binding, but it removes the practical replacement window for unprivileged
  actors. macOS immutability: `UF_IMMUTABLE` (owner/superuser changeable) / `SF_IMMUTABLE` (superuser,
  historically single-user-mode to clear); `fchflags(fd, …)` sets flags on an open fd. [Apple chflags(2)]
- macOS code signing / notarization / SIP do **not** by themselves bind an externally-supplied SHA-256
  to the executed bytes: a separate `codesign --verify path` + `execve(path)` is still two pathname
  resolutions. Kernel-enforced signature validation (Hardened Runtime) applies to loaded code but
  enforces the *signing policy*, not your particular digest allowlist. SIP does not protect an ordinary
  project path like `target/release/...`. [Apple TN3126; SIP guide]

### Immutable-artifact hardening (both platforms)

`chmod 0555` alone is insufficient (owner can re-chmod; anyone with write on the *parent dir* can
unlink/replace the name). Immutability must protect both file **contents** and the **directory entry**.
Linux: `chattr +i` / `FS_IMMUTABLE_FL` (needs `CAP_LINUX_IMMUTABLE`), or a root-owned non-writable
artifact dir. [man7 chattr(1), ioctl_iflags(2)]

### Applicability to `.factory/`

- The binary is `{project-root}/target/release/factory-dispatcher` — a **developer build output**, not
  a root-owned immutable artifact. On the same machine that builds it, a concurrent `cargo build` can
  legitimately replace it between the guard's hash and the shell's exec. This is the realistic F11
  threat here (not just an attacker).
- **Linux path:** the cleanest fix is a small trusted-launcher step or dispatcher subcommand that opens
  the binary, hashes the fd, and `fexecve`s the same fd — but note the migration binary IS
  factory-dispatcher, so the "launcher" and "target" are the same program; the practical move is to have
  the guard hand the *opened fd* (or an `execveat` on it) rather than a pathname to the shell.
- **macOS path (darwin-arm64):** since fexecve is unavailable, use protected staging: copy the verified
  binary to a fixed staging path the agent/build cannot rewrite mid-operation, set it read-only +
  `UF_IMMUTABLE` via `fchflags`, rehash, then execute the staged copy. Alternatively, **freeze the build
  before activation** (no concurrent `cargo build`) and verify under the maintenance lock immediately
  before exec, accepting residual TOCTOU as a documented, operationally-mitigated risk — this is a
  DECISION for the architect/human, and should be surfaced explicitly, not silently accepted.
- Because a fully-clean cross-platform fd-binding is not achievable on macOS with Apple-documented APIs,
  the architect should treat F11 as "reduce window + make replacement detectable" rather than "eliminate
  TOCTOU." The v1.3 language should stop implying the hash-then-exec-by-name design closes the window;
  it does not.

---

## RQ5 — Authorization / expiry during forward recovery (resolves F6; supports F5)

### The finding

v1.2 checks expiry immediately before PREPARED→COMMITTED (§4d) and mandates *deleting* PREPARED +
discarding staging on expiry — but that transition is placed **after** all target renames (§7c step 3).
So expiry can delete the recovery record **after** canonical content has already changed, stranding an
unrecoverable partial state. It also requires a fresh activation manifest after expiry while defining
activation_id as unique-per-activation, with **no transition that authorizes a new activation to
recover the old transaction.**

### Recommended pattern: authorization gates ENTRY to the irreversible phase; never abandon past the commit point

**Treat approval/authorization expiry as a gate on *entering* the irreversible phase — not as
permission to abandon work after the commit point.** Once a transaction crosses its commit point
(pivot), it must be driven to completion (forward recovery) or compensated — not aborted-by-deletion.
[Azure/AWS saga docs; 2PC; ARIES]

1. **Point of no return / commit point.** Saga literature calls the pivot the point of no return: after
   it succeeds, prior compensations are irrelevant and all following steps must be retryable and
   completed. Irreversible operations should occur only *after* critical validations succeed. In 2PC, a
   participant that has voted/prepared cannot unilaterally abort; the coordinator resolves in-doubt
   transactions and only forgets them after delivering + ack'ing the outcome. Oracle retains prepared
   transactions + locks and auto-resolves so all nodes reach the same outcome. **Design rule:** validate
   authorization BEFORE — ideally atomically with — the durable transition across the pivot:
   `validate approval → freeze manifest → durably record COMMITTING decision → begin irreversible
   action`. Before the pivot, expiry may abort/clean up. After it, expiry must NOT change the decision.
   [Azure saga; MS-DTCO 2PC; Oracle distributed txn]
2. **Leases + fencing tokens.** Kleppmann shows a holder can pause past its lease, another can acquire
   it, and the old holder can resume and corrupt state — so checking the lease just before writing is
   insufficient. Remedy: a strictly monotonic **fencing token** on every mutation, enforced at the
   resource, which rejects stale tokens. Chubby uses lease + **sequencers** (monotonic generation)
   validated by the receiving server. **Mapping:** an expired worker must not finish merely because it
   once held approval; a *recovery* worker gets a NEW fence and completes only the remaining steps of the
   already-committed transaction. The fence says *which worker is current*; the immutable transaction
   record says *what it may complete.* [Kleppmann 2016; Chubby OSDI'06]
3. **Recovery re-authorization without a fresh mutation right.** Issue a **completion-only recovery
   approval**, bound to `(old txn id, frozen manifest hash, staged-output hashes, original decision
   record, allowed remaining step ids)`, accepted only when the txn is already COMMITTING/IN_DOUBT/
   RECOVERING. It must reject: creating a new transaction, changing parameters, regenerating staged
   output, expanding scope, or invoking pre-pivot mutation APIs. Use stable idempotency keys
   `txn-id/step-id` + current fence. AWS idempotent-API guidance: store the request id + original params
   durably, treat retries as the same op, reject reuse with different params. [AWS Builders' Library;
   AWS Well-Architected REL]
4. **Saga / compensation.** When rollback is impossible, forward-recover with compensation; retain the
   transaction journal, step receipts, output hashes, idempotency results, and compensation plan until
   terminal COMPLETED/COMPENSATED. Compensation of a real, irreversible effect is a new corrective action
   — never deletion of the evidence that it happened. [Azure compensating-transaction; AWS saga]
5. **The anti-pattern (exactly F6).** Never let an approval-TTL cleanup delete a transaction that has
   reached COMMITTING/IN_DOUBT/RECOVERING/COMPENSATING — it destroys the evidence needed to complete,
   dedupe, reject altered intent, or compensate. Oracle keeps in-doubt records + locks until resolution;
   ARIES removes transaction state only after commit/rollback reaches its end record. Expiry may delete
   only an *unused pre-pivot* authorization after proving no irreversible step occurred. Post-pivot
   records get marked `ORIGINAL_APPROVAL_EXPIRED` while remaining recoverable; their lifetime is governed
   by transaction resolution + audit retention, NOT the approval TTL. [Oracle; ARIES]

### Recommended state machine (adapt v1.2 markers to this)

| State | Effect of original-approval expiry | Permitted next actions |
|---|---|---|
| AUTHORIZED / STAGING (pre-pivot) | stop new steps; verify no pivot effect occurred | abort + clean provisional state, or get an ordinary new approval |
| READY_TO_COMMIT | do not cross pivot unless valid at the atomic decision check | abort, or new ordinary approval before committing |
| COMMITTING / IN_DOUBT (post-pivot) | record expiry for audit; **do NOT reverse or delete** | recover decision; complete frozen manifest with a completion-only approval + current fence |
| COMPENSATING | expiry does not cancel compensation | resume idempotent compensation or escalate to human |
| COMPLETED / COMPENSATED | none | retain immutable outcome + audit; purge only per retention policy |

The specific "new approval bound to txn id + immutable output hashes" construction is a **governance
synthesis**, not a named standard — flagged — but it follows established 2PC-recovery, saga-pivot,
fencing, and idempotent-intent principles.

### Applicability to `.factory/`

- The pivot in ADR-052 is **the first rename** (first destructive publication step). v1.2's expiry
  recheck at §4d sits *after* the renames — wrong side of the pivot. **Move the last authorization check
  to immediately before the first rename, and once the first rename occurs, transition to a COMMITTING
  state whose recovery record is retained regardless of expiry.** This is the direct F6 fix.
- If publication becomes a single atomic pointer swap (RQ1), the pivot collapses to **one** instant (the
  pointer rename). Authorization is checked immediately before that single rename; there is no multi-step
  window in which expiry can strand a half-published set. **This is the strongest reason to adopt the
  atomic-pointer model: it makes F6 nearly trivial** (a single point-of-no-return instead of N).
- Recovery re-authorization maps to a state-manager-written **completion-only manifest** referencing the
  old `activation_id` + the staged-generation hash, distinct from a fresh F4 activation. The `activation_id`
  uniqueness constraint is satisfied by the recovery manifest *referencing* (not reusing) the original id.

---

## macOS/APFS durability specifics (cross-cuts RQ1, RQ2 — darwin-arm64 is a first-class target)

Verified against Apple's fsync(2) man page (Keith Smiley xcode-man-pages mirror + Apple archived
ManPages) and Apple fcntl(2):

1. **`fsync(2)` is NOT sufficient for power-loss durability on macOS.** Apple's fsync(2): *"while
   fsync() will flush all data from the host to the drive (i.e. the 'permanent storage device'), the
   drive itself may not physically write the data to the platters for quite some time."* [Apple fsync(2)]
2. **`F_FULLFSYNC` is required for the strongest guarantee.** Apple fsync(2)/fcntl(2): *"F_FULLFSYNC
   fcntl asks the drive to flush all buffered data to permanent storage. Applications, such as databases,
   that require a strict ordering of writes should use F_FULLFSYNC."* It is still not absolute (Apple
   notes some FireWire drives ignored the request). SQLite exposes this via `PRAGMA fullfsync` (default
   OFF) — so SQLite's normal sync on macOS should NOT be assumed to be a drive-cache flush. [Apple
   fsync(2), fcntl(2); sqlite.org PRAGMA]
3. **`F_BARRIERFSYNC` is an ordering barrier, weaker than F_FULLFSYNC for durability.** It does the
   fsync equivalent + issues an I/O barrier ordering earlier flushed writes before later I/O, but can
   return before earlier data reaches permanent media. Use it for crash-consistent ordering, not for a
   "all earlier writes are durable now" guarantee. (Documented in fcntl(2), not fsync(2).) [Apple fcntl(2)]
4. **Directory fsync on APFS — NOT VERIFIABLE against Apple docs (FLAGGED).** Apple's fsync(2)/fcntl(2)
   do NOT document that opening an APFS directory and calling fsync makes a rename durable, and do not
   promise Linux-equivalent directory-fsync semantics. **Do not present Linux-style APFS directory-fsync
   durability as an Apple-documented guarantee.** The ADR's Amendment 6 (mandatory dir-fsync per Pillai
   OSDI'14) is sound *on Linux*; on APFS it must be treated as best-effort and validated by testing, with
   F_FULLFSYNC on the file as the actually-documented durability lever. This is the single most important
   macOS caveat for v1.3.
5. **`rename(2)` atomic-replace IS supported on APFS** (namespace atomicity, same as Linux — old-or-new,
   never absent midway). `renamex_np` with `RENAME_SWAP` atomically *exchanges* two existing entries
   (both must exist) — useful for atomic pointer swap where you want the old pointer preserved. Namespace
   atomicity is still separate from durability. [Apple rename(2) semantics; renamex_np]
6. **Practical Rust guidance (write-temp → sync → rename → dir-sync):** on macOS replace the file-level
   `fsync` with `fcntl(fd, F_FULLFSYNC)` (or `F_BARRIERFSYNC` where only ordering is needed) before the
   rename; treat the parent-directory fsync as best-effort (call it, but do not rely on it for APFS
   durability — flagged). On Linux keep `fsync(file)` + `fsync(dir)`. A cross-platform abstraction should
   branch: `#[cfg(target_os="macos")]` → F_FULLFSYNC; else fsync + dir-fsync.

---

## Recommended v1.3 direction (synthesis for the architect)

1. **Replace per-file in-place rename with single-atomic-pointer publication over an immutable
   generation.** Build the complete new shard set in a staging generation, durably sync it, then publish
   with ONE atomic rename of a manifest/pointer (prefer a committed manifest file over a directory
   symlink, given the git-worktree context). This single change dissolves F1, F2, F5, F9 and collapses
   F6's multi-step pivot to a single point-of-no-return. Readers resolve the pointer once and pin one
   generation for the whole operation, replacing the ambiguous "COMMITTED absent ⇒ read legacy" heuristic.
2. **If fixed canonical shard paths must be retained,** at minimum introduce a **durable terminal-state
   manifest** as the single authoritative "which layout is live" record (resolving F9's steady-state
   ambiguity), and make the multi-file swap driven by one manifest write rather than N reader-visible
   in-place renames.
3. **Upgrade the phase marker to a framed, checksummed intent log** recording per-target expected
   post-content hash + expected pre-state before the first rename, with idempotent forward recovery that
   **accepts a matching destination hash as completion** and **fails closed on every ambiguous state**
   (F2). Specify durable ordering (staging→journal→rename→dir-sync→DONE) and mandate fault-injection
   tests between every step (ALICE / dm-log-writes).
4. **Replace the `O_CREAT|O_EXCL` PID-liveness-unlink lock with an advisory `flock` (Linux: or whole-file
   OFD lock) on a stable, never-unlinked lock inode** (F4). Treat the JSON body as diagnostic-only. This
   makes stale reclamation automatic (lock vanishes on process death) and eliminates the split-ownership
   race. Fail-closed on empty/corrupt metadata by *trying to acquire the lock first*, never by unlinking.
5. **Separate ephemeral OS-process ownership (the flock) from persistent maintenance intent (a durable
   txn record with a fencing generation)** (F5). Ordinary writers stay blocked for any unresolved
   post-pivot transaction regardless of PID liveness; a distinct recovery-owner acquisition updates
   ownership without clearing intent. Releasing the lock must never delete the intent.
6. **Implement a real drain: an admission gate (OPEN/DRAINING) with writer reservations spanning
   PreToolUse→tool-completion** in the native admission gate, so the coordinator waits for in-flight
   admitted writers before snapshotting inputs (F3). Keep the fingerprint recheck as defense-in-depth,
   not as the drain mechanism. Do not rely on shared→exclusive flock upgrade (non-atomic).
7. **Move the final authorization/expiry check to immediately before the first (now single) destructive
   step, and once past it, transition to COMMITTING and RETAIN the recovery record regardless of expiry**
   (F6). Add a completion-only recovery re-authorization bound to the old activation_id + staged-generation
   hash + allowed remaining steps, with a monotonic fence — distinct from a fresh F4 activation.
8. **Stop claiming the hash-then-exec-by-name design closes the executable TOCTOU (F11).** On Linux, bind
   verify-to-execute via an opened fd (`fexecve`/`execveat AT_EMPTY_PATH`), optionally memfd-seal. On
   macOS, `fexecve` is unavailable (Apple-documented exec is pathname-only) — use protected
   staging-and-launch (copy to a non-agent-writable path, set read-only + UF_IMMUTABLE, rehash, exec the
   staged copy) OR freeze the build under the maintenance lock and accept a documented, operationally-
   mitigated residual window. Surface this as an explicit architect/human DECISION.
9. **Make all durability barriers platform-branched:** Linux `fsync(file)+fsync(dir)`; macOS
   `fcntl(F_FULLFSYNC)` on the file (fsync alone does NOT flush the drive cache per Apple docs), with
   directory-fsync treated as best-effort-only on APFS (unverifiable — flagged). This corrects ADR
   Amendment 6, which assumes Linux dir-fsync semantics apply on APFS.
10. **Guard-stack completeness (F7, F8, F10):** these are spec-gaps rather than crash-safety patterns and
    are answerable in-scope by the architect/devops without external research — conservative Bash
    admission (block unknown write effects under maintenance intent; classify sanctioned commands from
    config-driven target sets), separate guard branches for read-only census / terminal-no-op / new
    activation / recovery, and one authoritative allowlist covering top-level + subsystem manifests +
    sub-shards + staging/backup files, with the CLAUDE.md amendment generated from that same scope.

---

## Inconclusive / flagged items (be explicit)

- **APFS directory-fsync durability (HIGH importance, INCONCLUSIVE):** Apple's fsync(2)/fcntl(2) do NOT
  document that fsync on an APFS directory fd makes a rename power-loss-durable. Treat as best-effort;
  the documented macOS durability lever is `F_FULLFSYNC` on the file. Requires empirical testing on
  darwin-arm64 to characterize; do not assert Linux-parity.
- **`fexecve` on macOS (MEDIUM-HIGH importance, INDIRECTLY CONFIRMED):** No Apple document *explicitly*
  says "macOS does not implement fexecve." The conclusion rests on Apple's exec man-pages exposing only
  pathname-based APIs + corroborating non-Apple SDK/source reports. Confidence: high that it is absent,
  but not backed by an explicit Apple negative statement.
- **OFD lock availability on older macOS / network volumes (LOW-MEDIUM):** modern XNU headers document
  OFD locks, but Apple's archived fcntl(2) does not; runtime-test on the target macOS version and
  filesystem. `flock` is the safer cross-platform default.
- **Hash-as-completion-witness recovery table (design synthesis, not a POSIX guarantee):** follows
  established ext4/XFS/ARIES result-oriented idempotent-replay principles, but the specific hash decision
  matrix is application protocol and must be validated by fault injection.
- **Completion-only recovery re-authorization construction (governance synthesis):** not a named
  standard; derived from 2PC recovery + saga pivot + fencing + idempotent-intent literature.
- **RPM/dpkg as prior art for a full per-file rename-intent/done journal (NEGATIVE):** could NOT verify
  that current RPM or dpkg durably records every installed-file staging→target rename + expected post-hash
  before publication. dpkg's status-DB journal and the dpkg 2.0 filesystem-journal *proposal* are
  narrower/different evidence. Do not cite RPM/dpkg as equivalent to the proposed intent-log scheme.

---

## Research Methods

| Tool | Queries | Purpose |
|------|---------|---------|
| **Perplexity perplexity_research (PRIMARY)** | 6 | Atomic multi-file publication (1 timeout + 1 success retry); WAL/intent-log recovery; single-writer exclusion/advisory locking; executable verify-to-execute TOCTOU; authorization/expiry during forward recovery; APFS fsync semantics (timed out — see below) |
| Perplexity perplexity_ask | 1 | macOS/APFS F_FULLFSYNC / F_BARRIERFSYNC / directory-fsync / rename atomicity fast grounded lookup (fallback after research timeout) |
| Perplexity perplexity_reason | 0 | — |
| Perplexity perplexity_search | 0 | — |
| Context7 | 0 | Not needed — questions were OS/filesystem-API + systems-architecture patterns, best served by man-page-grounded Perplexity + direct Apple man-page fetch |
| WebFetch | 1 | Direct verbatim verification of Apple macOS fsync(2) man page (hardware-cache-flush + F_FULLFSYNC language) |
| WebSearch | 0 | — |
| Training data | ~0 areas | All non-obvious claims are web/man-page grounded; training knowledge used only to structure queries and cross-check |

**Total MCP tool calls:** 8 (7 Perplexity + 1 WebFetch). 2 `perplexity_research` calls timed out at
the 300s API limit (initial RQ1; APFS deep-dive) — RQ1 was re-run successfully at medium context; the
APFS deep-dive was recovered via `perplexity_ask` + a direct Apple-man-page `WebFetch`, which produced a
verbatim-quoted authoritative answer.

**Training data reliance:** low — every load-bearing OS-API claim (rename atomicity, fsync vs
F_FULLFSYNC, flock/fcntl/OFD semantics, fexecve availability, fencing tokens) is grounded in man7.org,
Apple Developer man pages, kernel.org, LWN, SQLite/LMDB/Postgres docs, the OSDI'14 Pillai paper, or
Kleppmann/Chubby/saga literature, with uncertain items explicitly flagged above.
