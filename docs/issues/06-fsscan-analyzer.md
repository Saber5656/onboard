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
   otherwise recursive `fs.readdir` walk applying `.gitignore` files at each level via
   the `ignore` package. Record which strategy ran in a warning-level note (`fsscan-strategy`).
2. Apply, in order: built-in excluded directory list (§6.2 item 2, matched on any path
   segment), sensitive matcher (item 3 — excluded entries are **not** listed in output at
   all), config `exclude` globs then `include` globs (picomatch-style via `ignore`;
   include filters the survivor set).
3. Symlinks: `lstat` every candidate; if symlink → skip content and listing, push warning
   `fsscan-symlink-skipped` with the relative path (target never resolved).
4. Binary detection (item 5): extension denylist first (content never read); else read
   first 8 KB — NUL byte ⇒ binary. Binary ⇒ `lang: null`. Hash rule: sha256 is computed
   for fully-read files only; binary/oversized files get `sha256 = ""` (empty string,
   documented) so downstream never depends on it.
5. Minified heuristic: text file with any single line > 5000 chars ⇒ treated like
   binary. No new model field is added for this: push warning `fsscan-minified` and set
   `lang: null` so excerpt logic skips the file.
6. Caps: file > `maxFileSizeKB` ⇒ listed, content unread, warning `fsscan-oversize`;
   after sorting the candidate list, keep first `maxFiles` entries, warning
   `fsscan-maxfiles` with dropped count.
7. `isTest` heuristics exactly §6.2 item 7. `docsFiles` detection: root-level
   `README*` (kind readme), `CONTRIBUTING*`, `LICENSE*`, `CODE_OF_CONDUCT*` (case-insensitive).
8. Output ordering: `files` sorted by `path` byte order; `languages` aggregated
   (files/bytes) sorted by `bytes` desc then lang asc. All paths via `toRepoRelPosix`
   + `stripControlChars`; reads via `safeJoin` only.
9. Never throw for content reasons; environment failure (root unreadable) →
   `AnalysisError("fsscan-root-unreadable")`.

## Acceptance Criteria

- [ ] mini-express-app: exactly the committed tree is listed (lockfile included), `.github/workflows/ci.yml` present, `languages` counts ts > 0, `isTest` true for the two test files, README detected.
- [ ] hostile fixture (materialized): `.env` absent from output; `escape-link` absent with `fsscan-symlink-skipped` warning; `big.bin` listed with empty sha256 + `fsscan-oversize`; `minified.js` gets `lang: null` + warning; `<img …>.ts` listed with control-char-free path; `docs/日本語-メモ.md` listed NFC-normalized.
- [ ] git-strategy and walk-strategy (same tree with `.git` removed) produce identical `files` output for js-lib (assert deep-equal minus the strategy warning).
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
