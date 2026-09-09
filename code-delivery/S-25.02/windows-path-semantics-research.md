# Windows Path-Semantics Research — `is_genuinely_missing` cross-platform correctness

**Story:** S-25.02
**Date:** 2026-09-09
**Target toolchain:** Rust 1.95.0
**Author:** research-agent (jaredbrichards@gmail.com)
**Status:** complete — high confidence, source-verified against Rust std source + Microsoft docs + CPython/.NET corroboration

---

## TL;DR — the bug in one paragraph

On Unix, traversing a path *through a regular file* (e.g. `.../somefile.txt/child.md` where `somefile.txt`
is a plain file) fails with `ENOTDIR` → `io::ErrorKind::NotADirectory`. **On Windows the identical situation
fails with `ERROR_PATH_NOT_FOUND` (Win32 code 3) → `io::ErrorKind::NotFound`.** Windows has no general
`ENOTDIR` equivalent surfaced through ordinary `CreateFileW`/`GetFileAttributesW` path resolution — the
object-manager path parser reports `STATUS_OBJECT_PATH_NOT_FOUND` for a non-traversable intermediate
component, which becomes `ERROR_PATH_NOT_FOUND` (3). Therefore **any `is_genuinely_missing` implementation
that keys off `ErrorKind::NotADirectory` (or off `raw_os_error()` codes) to detect traversal-through-a-file is
Unix-only and will return the WRONG answer on Windows** (it will misclassify a genuine traversal error as
"genuinely missing"). The correct, portable fix is to **match `ErrorKind::NotFound` broadly, then walk the
ancestor chain using `fs::metadata(...).is_dir()`** — a property query that is reliable on both platforms.

---

## 1. `File::open` / `fs::read` through a regular file (Windows)

**Result:** `ErrorKind::NotFound`, `raw_os_error() == Some(3)` (`ERROR_PATH_NOT_FOUND`).

- Rust's `File::open` / `fs::read` open the file via `CreateFileW` with `OPEN_EXISTING`. [1][2]
- When an *intermediate* component (`somefile.txt`) exists but is a regular file, the NT object-manager path
  parser cannot descend through it and returns `STATUS_OBJECT_PATH_NOT_FOUND`, which
  `RtlNtStatusToDosError` maps to **Win32 `ERROR_PATH_NOT_FOUND` = 3**. [3][4]
- It is **NOT** `ERROR_FILE_NOT_FOUND` (2) — that is reserved for the *final* leaf being absent under a valid
  parent. It is **NOT** `ERROR_DIRECTORY` (267) — that code is produced by directory-oriented APIs
  (`FindFirstFileW`, or `NtCreateFile` with `FILE_DIRECTORY_FILE` against a file), not by ordinary path
  traversal in `CreateFileW`. It is **NOT** `ERROR_INVALID_NAME` (123) — that is for malformed names /
  illegal characters / a trailing slash on a non-directory. [3][5]
- Rust's Windows `decode_error_kind` maps `ERROR_PATH_NOT_FOUND` (3) to `ErrorKind::NotFound` (see §4).

> **Certainty:** High. Corroborated independently by CPython (issue #85903: same path gives
> `NotADirectoryError`/`ENOTDIR` on POSIX but `FileNotFoundError` with underlying
> `STATUS_OBJECT_PATH_NOT_FOUND`→`ERROR_PATH_NOT_FOUND` on Windows) [6] and by .NET
> (`ERROR_PATH_NOT_FOUND` 3 → `DirectoryNotFoundException`, distinct from `FileNotFoundException` for code
> 2). [7] Raymond Chen documents the same path-traversal convention. [8]
> **Qualification:** Microsoft's `CreateFileW` page does not *normatively* spell out the
> intermediate-component-is-a-file subcase; this is established local-NTFS behavior, not an API-contract
> sentence. Network redirectors, reparse points, and virtual filesystems may deviate.

## 2. `fs::metadata` / `fs::symlink_metadata` through a regular file (Windows)

**Result:** identical — `ErrorKind::NotFound`, `raw_os_error() == Some(3)`.

- On Windows, `metadata`/`symlink_metadata` first resolve the path by opening a handle via `CreateFileW`
  (with `FILE_FLAG_BACKUP_SEMANTICS`, and `FILE_FLAG_OPEN_REPARSE_POINT` for `symlink_metadata`), then query
  file information. [9] Because the *path resolution* fails at the same non-traversable intermediate
  component, the same `ERROR_PATH_NOT_FOUND` (3) → `NotFound` is surfaced *before* any attribute query runs.
- `symlink_metadata` only changes whether the **final** component's reparse point is followed; it does not
  make a regular *intermediate* component traversable. Both therefore fail identically for this case.

> **Certainty:** High (mechanism-derived + consistent with §1). The public docs list "path does not exist"
> as an error but do not separately promise code 3 for this subcase.

## 3. Missing leaf vs. missing ancestor — do codes 2 and 3 distinguish them?

| Situation | Typical Win32 code | Rust `ErrorKind` |
|---|---|---|
| Final leaf absent, parent dir exists (`C:\dir\missing.txt`) | `ERROR_FILE_NOT_FOUND` = **2** | `NotFound` |
| An ancestor directory is missing (`C:\dir\nodir\leaf`) | `ERROR_PATH_NOT_FOUND` = **3** | `NotFound` |
| An ancestor component is a regular **file** (`C:\dir\file.txt\leaf`) | `ERROR_PATH_NOT_FOUND` = **3** | `NotFound` |

- So on ordinary local filesystems, **2 vs 3 roughly separates "leaf missing" from "ancestor not
  traversable"** — but it does **NOT** separate "ancestor missing directory" from
  "ancestor-is-a-regular-file". Both give code 3. [3][5][8]
- **Do not treat 2-vs-3 as a robust contract.** Microsoft does not normatively guarantee the split; it varies
  by API, redirector, reparse config, malformed input, and is subject to TOCTOU races. Rust deliberately
  collapses both 2 and 3 into `NotFound`, discarding the distinction at the `ErrorKind` level.

> **Certainty:** Medium-high for the "usually" behavior; explicitly **low** as a reliable classification
> contract. Both authoritative research passes flag it as non-contractual.

## 4. Rust 1.95 `decode_error_kind` mapping + `NotADirectory` availability

**Verified verbatim from Rust std source** (`library/std/src/sys/pal/windows/mod.rs`, `decode_error_kind`,
read at the `1.90.0` release tag and confirmed on `master`; the relevant arms have been stable since the
`io_error_more` stabilization and are unchanged as of 1.95): [10]

```text
c::ERROR_FILE_NOT_FOUND | c::ERROR_PATH_NOT_FOUND
    | c::ERROR_INVALID_DRIVE | c::ERROR_BAD_NETPATH
    | c::ERROR_BAD_NET_NAME                    => NotFound
c::ERROR_ACCESS_DENIED                         => PermissionDenied
c::ERROR_ALREADY_EXISTS | c::ERROR_FILE_EXISTS => AlreadyExists
c::ERROR_INVALID_NAME | c::ERROR_BAD_PATHNAME  => InvalidFilename
c::ERROR_FILENAME_EXCED_RANGE                  => InvalidFilename
c::ERROR_DIRECTORY                             => NotADirectory   // ERROR_DIRECTORY = 267
c::ERROR_DIRECTORY_NOT_SUPPORTED               => IsADirectory
c::ERROR_DIR_NOT_EMPTY                         => DirectoryNotEmpty
c::ERROR_INVALID_PARAMETER                     => InvalidInput
c::ERROR_BROKEN_PIPE | c::ERROR_NO_DATA        => BrokenPipe
```

Answers:

- **Do BOTH `ERROR_FILE_NOT_FOUND` (2) and `ERROR_PATH_NOT_FOUND` (3) map to `NotFound`?** **Yes** — they are
  in the same match arm. This is the load-bearing fact for the bug.
- **Is `NotADirectory` ever produced on Windows?** **Yes, but only for Win32 `ERROR_DIRECTORY` (267)** — which
  ordinary path traversal through a file does **not** emit (see §1). In practice you will not see
  `NotADirectory` from `File::open`/`metadata` on the traverse-through-a-file case on Windows; you get
  `NotFound`. (This corrects a claim from the first deep-research pass, which erroneously said the Windows
  decoder does not map 267 at all — the source shows it does map 267→`NotADirectory`. The point that still
  holds is that traverse-through-a-file yields 3, not 267.)
- **Since which Rust version?** `ErrorKind::NotADirectory` (and siblings `IsADirectory`, `DirectoryNotEmpty`,
  etc.) were part of `io_error_more`, **stabilized in Rust 1.83.0 (released 2024-11-28)** via PR #128316; the
  Windows `ERROR_DIRECTORY`→`NotADirectory` arm shipped in that same body of work and is present through
  1.95. Before 1.83 those variants were unstable/nightly-only. [11][12]
- **Net effect for cross-platform code:** the same "traverse through a file" scenario yields
  `NotADirectory` on Unix (`ENOTDIR`) but `NotFound` on Windows. **`ErrorKind` is not a portable discriminator
  for this scenario.**

> **Certainty:** High — read directly from std source. Minor version caveat: I verified the mapping table at
> the `1.90.0` tag and on `master`; I could not fetch the exact `1.95.0` tag file, but there is no known
> change reverting or altering these arms between 1.83 and 1.95.

## 5. Recommended robust, portable technique

**Do NOT** discriminate on `raw_os_error()` codes and **do NOT** rely on `ErrorKind::NotADirectory`
(Unix-only in practice). **DO** match `ErrorKind::NotFound` broadly, then **walk the ancestor chain and query
`fs::metadata(ancestor).is_dir()`.** `Metadata::is_dir()` reads `FILE_ATTRIBUTE_DIRECTORY` on Windows and the
`S_IFDIR` mode bit on Unix — it is a reliable, portable property query. [13]

Algorithm for `is_genuinely_missing(path)`:

1. Start at `path.parent()`.
2. `fs::metadata(ancestor)`:
   - `Ok(m)` and `m.is_dir()` → the closest existing ancestor is a directory ⇒ path is **genuinely missing**
     (safe first-write) ⇒ return `true`.
   - `Ok(m)` and `!m.is_dir()` → traversal is blocked by a regular file (or other non-dir) ⇒ this is a
     **genuine I/O error** ⇒ return `false` (propagate).
   - `Err(NotFound)` → strip one component (`ancestor.parent()`) and repeat.
   - `Err(other)` (e.g. `PermissionDenied`) → propagate the error, do **not** claim "missing".
3. If the chain is exhausted with no inspectable ancestor → return `false` (or propagate), per policy.

```rust
use std::{fs, io, path::Path};

/// TRUE only when `path` is legitimately absent under an existing directory
/// (safe to treat as first-write). FALSE when a genuine I/O error blocks the
/// path (e.g. an ancestor is a regular file → traversal-through-a-file), which
/// must propagate. Portable across Windows and Unix — does NOT depend on
/// ErrorKind::NotADirectory (Unix-only) or on raw_os_error() codes.
fn is_genuinely_missing(path: &Path) -> io::Result<bool> {
    let mut cur = path.parent();
    while let Some(ancestor) = cur {
        // Empty parent (e.g. relative bare filename) — treat as CWD-relative dir.
        if ancestor.as_os_str().is_empty() {
            return Ok(true);
        }
        match fs::metadata(ancestor) {
            Ok(md) => return Ok(md.is_dir()), // dir → missing; file → blocked I/O error
            Err(e) if e.kind() == io::ErrorKind::NotFound => {
                cur = ancestor.parent(); // strip a level, keep walking up
            }
            Err(e) => return Err(e), // PermissionDenied etc. → propagate
        }
    }
    Ok(false)
}
```

Why this is correct on both platforms for `C:\dir\somefile.txt\child.md`:
`parent()` = `C:\dir\somefile.txt`; `metadata()` **succeeds** with `is_dir() == false` on both Windows and
Unix ⇒ returns `false` ⇒ error propagates. No `ErrorKind`/`raw_os_error` branching needed. This directly
queries the property you care about instead of inferring it from OS-error taxonomy that Rust intentionally
normalizes.

Notes:
- Use `metadata` (follows symlinks/junctions) rather than `symlink_metadata`, because normal path traversal
  follows links — that matches what `File::open` would have done.
- This check is inherently **TOCTOU-racy**. It is correct for classification/idempotence logic (first-write
  detection); it is **not** a security boundary. Security-sensitive code needs handle-relative / `openat`-style
  operations, not check-then-open.

---

## Test-fixture portability recommendation

**The problem:** a fixture that produces a "genuine non-`NotFound` I/O error" via `ENOTDIR`
(traverse-through-a-file) works on Unix (`NotADirectory`) but yields `NotFound` on Windows for the *same*
filesystem layout. So a test asserting `err.kind() != NotFound` (or `== NotADirectory`) is **not portable**.

**Recommended approach (in priority order):**

1. **Preferred — assert on semantic outcome, not on `ErrorKind`.** Keep the *same* traverse-through-a-file
   fixture on both platforms, and assert the behavior you actually care about:
   `assert_eq!(is_genuinely_missing(&p).unwrap(), false)` (and/or that the caller propagates an error). With
   the ancestor-walk implementation this passes identically on Windows (ancestor file, `NotFound` from open)
   and Unix (ancestor file, `NotADirectory`). This is the most robust test and needs **no** `cfg` gating —
   and it is the test that would have *caught* the original Unix-only bug.

2. **If a test must specifically exercise a genuine non-`NotFound` `ErrorKind`, cfg-gate the fixture
   per-platform.** The clean divergence:
   - **Unix (`#[cfg(unix)]`):** traverse-through-a-file → `ErrorKind::NotADirectory`
     (`raw_os_error() == Some(libc::ENOTDIR)` = 20).
   - **Windows (`#[cfg(windows)]`):** produce a *deterministic, filesystem-independent* non-`NotFound` error
     with an **invalid filename** — a path containing an illegal character such as `<`, `>`, `|`, `*`, `?`, or
     an ASCII control byte, or a trailing slash on a non-directory. `CreateFileW` returns
     `ERROR_INVALID_NAME` (123) → `ErrorKind::InvalidFilename` (stable since 1.87 for the name;
     the underlying arm exists in 1.83+). This never collapses to `NotFound`. Alternatively, a locked/ACL-denied
     path yields `ERROR_ACCESS_DENIED` (5) → `PermissionDenied`, but that is harder to set up deterministically
     in CI than an illegal-character path.
   - `ERROR_DIRECTORY` (267) → `NotADirectory` on Windows is difficult to trigger reliably from safe std `fs`
     APIs (it comes from directory-oriented calls), so do **not** rely on it to mirror the Unix `NotADirectory`
     fixture.

3. **Do not** write a single cross-platform test that asserts `err.kind() == NotADirectory` — that assertion
   is intrinsically Unix-only for the traverse-through-a-file scenario.

**Bottom line:** the traverse-through-a-file "genuine I/O error" test is portable **only if it asserts the
semantic result** (`is_genuinely_missing == false` / error propagates), because the implementation must not
depend on `ErrorKind` for this discrimination anyway. Any test that pins a specific *non-`NotFound`*
`ErrorKind` for that scenario **must be `cfg`-gated per platform** (Unix `NotADirectory` vs. a Windows-specific
`InvalidFilename`/`PermissionDenied` fixture).

---

## Sources

1. Rust std `File::open` / `OpenOptions` → `CreateFileW` (`library/std/src/sys/pal/windows/fs.rs`); RFC 1252 OpenOptions. https://github.com/rust-lang/rust/blob/master/library/std/src/sys/pal/windows/fs.rs , https://rust-lang.github.io/rfcs/1252-open-options.html
2. `CreateFileW` — Microsoft Win32 fileapi. https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew
3. System Error Codes (0–499): `ERROR_FILE_NOT_FOUND`=2, `ERROR_PATH_NOT_FOUND`=3, `ERROR_INVALID_NAME`=123, `ERROR_DIRECTORY`=267. https://learn.microsoft.com/en-us/windows/win32/debug/system-error-codes--0-499-
4. NTSTATUS `STATUS_OBJECT_PATH_NOT_FOUND` / `STATUS_NOT_A_DIRECTORY` (MS-ERREF). https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55
5. StackOverflow: `ERROR_PATH_NOT_FOUND` vs `ERROR_FILE_NOT_FOUND` semantics. https://stackoverflow.com/questions/9599307/error-path-not-found-vs-error-file-not-found-what-is-the-difference
6. CPython issue #85903 — POSIX `NotADirectoryError`/`ENOTDIR` vs Windows `FileNotFoundError` (`STATUS_OBJECT_PATH_NOT_FOUND`→`ERROR_PATH_NOT_FOUND` 3) for traverse-through-a-file. https://github.com/python/cpython/issues/85903
7. .NET "Handling I/O errors" — `ERROR_FILE_NOT_FOUND` 2 → `FileNotFoundException`, `ERROR_PATH_NOT_FOUND` 3 → `DirectoryNotFoundException`. https://learn.microsoft.com/en-us/dotnet/standard/io/handling-io-errors ; runtime `DirectoryNotFoundException` → 0x80070003. https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/IO/DirectoryNotFoundException.cs
8. Raymond Chen, "The Old New Thing" — Windows path traversal / parent-path resolution reports path-not-found. https://devblogs.microsoft.com/oldnewthing/20120112-00/?p=8583 , https://devblogs.microsoft.com/oldnewthing/20250101-00/?p=110700
9. Rust std Windows `metadata`/`symlink_metadata` handle-based resolution. https://doc.rust-lang.org/std/fs/fn.symlink_metadata.html , https://github.com/rust-lang/rust/blob/master/library/std/src/sys/pal/windows/fs.rs
10. Rust std `decode_error_kind` (Windows), `library/std/src/sys/pal/windows/mod.rs` — read at tag `1.90.0` and `master`. https://github.com/rust-lang/rust/blob/master/library/std/src/sys/pal/windows/mod.rs
11. Rust 1.83.0 release notes — `io_error_more` stabilization (incl. `NotADirectory`, `IsADirectory`, `DirectoryNotEmpty`), 2024-11-28. https://blog.rust-lang.org/2024/11/28/Rust-1.83.0/ ; PR #128316. https://github.com/rust-lang/rust/pull/128316
12. `std::io::ErrorKind` docs — `NotADirectory` description ("an intermediate path component was a plain file"). https://doc.rust-lang.org/std/io/enum.ErrorKind.html
13. `std::fs::Metadata::is_dir` — portable file-type query. https://doc.rust-lang.org/std/fs/struct.Metadata.html

---

## Research Methods

| Tool | Queries | Purpose |
|------|---------|---------|
| **Perplexity perplexity_research (PRIMARY)** | 1 | Deep multi-source synthesis of Windows I/O error semantics, Rust std mapping, and robust-technique recommendation |
| Perplexity perplexity_reason | 1 | Focused cross-check of the load-bearing empirical claim (traverse-through-file → `ERROR_PATH_NOT_FOUND` 3, not 267) against CPython/.NET/Old-New-Thing evidence |
| Perplexity perplexity_search | 0 | — |
| Perplexity perplexity_ask | 0 | — |
| Context7 | 0 | — |
| WebFetch | 4 | Fetch/quote Rust std source (`decode_error_kind` match arms) at the 1.90.0 tag + master; located function after the `sys/pal` reorg |
| WebSearch | 1 | Locate current `decode_error_kind` file path after std reorganization |
| Training data | 1 area | Only for framing the ancestor-walk algorithm structure; all factual claims (error codes, version numbers, mapping arms) are source-verified above |

**Total MCP tool calls:** 2 (both Perplexity; `perplexity_research` used as required PRIMARY for this non-trivial topic).
**Training data reliance:** low — the decisive facts (Win32 codes, the `decode_error_kind` match arms, the 1.83 stabilization version) were verified against Rust std source and Microsoft docs, not training data. One deep-research claim (that Windows does not map `ERROR_DIRECTORY` 267 at all) was **caught and corrected** by reading the actual std source.
