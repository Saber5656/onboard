# Title

fs-scan analyzer: file listing, exclusions, caps, language map

## Summary

Implement `src/analyze/fsscan/` per DESIGN §6.2: enumerate repo files (git-aware or
walk-based), apply built-in + sensitive + config exclusions, detect binaries, enforce
size/count caps, classify languages and test files, and emit sorted `FileNode[]` with
hashes plus `languages` stats and `docsFiles`.

## Context

First pipeline stage; every other analyzer consumes its output. It is also a security
boundary: sensitive files must never be read (T2), symlinks never followed (T1).

## Scope

- `src/analyze/fsscan/index.ts` — `scanFiles(root, config, logger) → { files, languages, docsFiles, warnings }`.
- `src/analyze/fsscan/lang-map.ts` — extension→language table (§6.2 item 7) + binary-extension denylist.
- `src/analyze/fsscan/sensitive.ts` — sensitive-file matcher (§6.2 item 3).
- Unit tests against fixtures (05).

## Detailed Requirements

1. Listing: when `<root>/.git` exists **and** `git` executes successfully, use
   `git ls-files --cached --others --exclude-standard -z` (cwd=root, parse NUL-separated);
   candidates that no longer exist on disk (`ENOENT` on lstat — e.g. tracked-but-deleted
   in a dirty worktree) are skipped with warning `fsscan-missing-skipped`. Otherwise:
   recursive `fs.readdir` walk applying `.gitignore` semantics via the `ignore` package —
   maintain a root-to-leaf stack of `ignore` instances, each `.gitignore`'s patterns
   evaluated relative to its own directory, `!` negations supported per the package;
   directories pruned when ignored. Record which strategy ran in a warning-level note
   (`fsscan-strategy`).
2. Apply, in order: built-in excluded directory list (§6.2 item 2, matched on any path
   segment), sensitive matcher (item 3 — excluded entries are **not** listed in output
   at all), then config patterns. Config `exclude`/`include` use **gitignore syntax via
   the `ignore` package** over repo-relative POSIX paths (not picomatch): `exclude`
   entries feed one ignore instance that removes matches; when `include` differs from
   the default `["**/*"]`, a second ignore instance built from the include list acts as
   an allowlist — keep only paths it matches.
3. Symlinks: `lstat` every candidate; if symlink → skip content and listing, push warning
   `fsscan-symlink-skipped` with the relative path (target never resolved).
   Path handling for hostile names: the **raw** relative path is used for all disk I/O
   (`safeJoin`, `lstat`, reads); the **sanitized** path (`toRepoRelPosix` +
   `stripControlChars`) is what enters `FileNode.path`. If two raw paths sanitize to the
   same output path, keep the byte-order-first one and warn `fsscan-path-collision`.
4. Size is taken from `lstat` for every listed regular file **before** any content
   decision; the oversize check (item 6) therefore applies to binary-denylisted files
   too (e.g. `big.bin` gets `fsscan-oversize` even though its content would never be
   read anyway). Binary detection (item 5): extension denylist first (content never
   read); else read first 8 KB — NUL byte ⇒ binary. Binary ⇒ `lang: null`. Hash rule:
   sha256 is computed for fully-read files only; binary/oversized/minified files get
   `sha256 = ""` (the documented sentinel, DESIGN §5.2) so downstream never depends on it.
   During the same full read, `lineCount` is computed (`\n` count + 1 for a non-empty
   final line; 0 for empty file); files never fully read get `lineCount: null`
   (module-mapper loc input, §6.5).
5. Minified heuristic: text file with any single line > 5000 chars ⇒ treated like
   binary. No new model field is added for this: push warning `fsscan-minified` and set
   `lang: null` so excerpt logic skips the file.
6. Caps: file > `maxFileSizeKB` ⇒ listed, content unread, warning `fsscan-oversize`;
   after sorting the candidate list, keep first `maxFiles` entries, warning
   `fsscan-maxfiles` with dropped count.
7. `isTest` heuristics exactly §6.2 item 7. `docsFiles` detection: root-level
   `README*` (kind readme), `CONTRIBUTING*`, `LICENSE*`, `CODE_OF_CONDUCT*` (case-insensitive).
8. Output ordering: `files` sorted by `path` byte order; `languages` aggregates **only
   files with `lang !== null`** (files count + bytes from `FileNode.size`), sorted by
   `bytes` desc then lang asc.
9. Never throw for content reasons; environment failure (root unreadable) →
   `AnalysisError("fsscan-root-unreadable")`.

## Acceptance Criteria

- [ ] mini-express-app: exactly the committed tree is listed (lockfile included), `.github/workflows/ci.yml` present, `languages` counts ts > 0, `isTest` true for the two test files, README detected.
- [ ] hostile fixture (materialized; POSIX-only test, skipped on win32): `.env` absent from output; `escape-link` absent with `fsscan-symlink-skipped` warning; `big.bin` listed with empty sha256 + `fsscan-oversize`; `minified.js` gets `lang: null` + `fsscan-minified` warning; `<img …>.ts` listed with control-char-free path; `docs/日本語-メモ.md` listed NFC-normalized.
- [ ] git-strategy and walk-strategy (same tree with `.git` removed) produce identical `files` output for js-lib (assert deep-equal minus the strategy warning); a nested-`.gitignore`-with-negation case is included in this parity test.
- [ ] Dirty-worktree case: delete a tracked file without committing → it is skipped with `fsscan-missing-skipped`, no throw.
- [ ] `maxFiles: 5` config on mini-express-app keeps the 5 byte-order-first paths and warns with the correct dropped count.
- [ ] No test reads `/etc` or escapes the temp dir (safeJoin enforced — code review checklist item).

## Validation

`pnpm --filter onboard-cli-placeholder test` (fsscan suite); run twice → identical output
(determinism spot-check on this stage's serialized output).

## Dependencies

02, 05.

## Non-goals

Import edges, symbols (09–11); secret content scanning (23 — fs-scan only excludes by
*name*); watch mode.

## Design References

DESIGN §6.2, §5.2 FileNode, §11.2 T1/T2, §13 caps.
