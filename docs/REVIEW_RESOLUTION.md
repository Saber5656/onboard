# Review resolution contract

This addendum is a documentation-only acceptance contract for PR #42. It records the resolution required for each existing review thread. It does not claim that product implementation or tests have been completed. The existing Bot review is the sole Bot input for this PR and will not be retriggered.

## PRRT_kwDOTNkCxM6QARrD — allowlist IDs bind to findings without storing secrets

Finding: An allowlist identifier based only on a line or position can collide for multiple secrets on the same line and does not bind the approval to the matched value.

Normative resolution:
- Derive the allowlist ID from the finding's stable location (including the relevant offset/span) and a one-way hash of the matched secret or equivalent non-reversible fingerprint.
- Never persist or expose the raw secret in the ID, report, or allowlist record.
- Duplicate findings at the same line must remain distinguishable, and an allowlist entry must not authorize a different value or shifted span.

Focused verification before resolving this thread:
- Create two different secrets on one line and assert their IDs differ while neither ID reveals the raw value.
- Modify the secret or its span and assert the prior allowlist entry no longer matches.
- Inspect logs, serialized reports, and allowlist files for raw-secret leakage.

## PRRT_kwDOTNkCxM6QARrJ — Shiki output is compatible with style-src

Finding: Shiki inline style attributes conflict with a style-src self policy unless the CSP explicitly allows inline styles.

Normative resolution:
- Prefer class-based tokens plus repository-controlled CSS or a CSS transformer that emits external styles.
- Do not weaken style-src with unsafe-inline solely to accommodate the renderer.
- If an inline-style exception is ever proposed, it requires a separately reviewed CSP decision and must not be implied by this PR.

Focused verification before resolving this thread:
- Render representative Shiki output and assert generated HTML has no disallowed inline style attributes.
- Serve the viewer with the declared style-src policy and verify syntax highlighting remains functional.
- Test both supported themes and confirm CSS classes are deterministic and self-contained.

## PRRT_kwDOTNkCxM6QARrM — secret scan exclusions are repository-controlled

Finding: --exclude-standard includes global and user info/exclude rules, making a scan dependent on ambient machine configuration.

Normative resolution:
- The secret scan must use only repository-controlled ignore sources declared by the scan policy.
- Global excludes, user info/exclude files, and ambient environment configuration must not change the scan result.
- Any explicit generated/vendor exclusions must be versioned, auditable, and reported in the scan metadata.

Focused verification before resolving this thread:
- Run the scan with different global/info exclude files and assert identical results.
- Add and remove a repository-controlled ignore entry and assert only that declared change affects the result.
- Record the effective ignore-source list and fail when an undeclared source is consulted.

## PRRT_kwDOTNkCxM6QARrP — serialized paths and names enter the secret scan

Finding: Scanning only file contents misses secrets embedded in serialized paths, module names, and repository metadata.

Normative resolution:
- The scan input must include serialized repository paths, module/package names, and repoName fields alongside file contents.
- Normalize these values according to one documented encoding before matching, without dropping separators or meaningful case.
- Findings must identify the input component while continuing to redact matched secret material.

Focused verification before resolving this thread:
- Place a secret-like value in a path, module name, repoName, and file body and assert each is scanned.
- Confirm serialized input is stable across platforms and does not leak raw matches in diagnostics.
- Test empty, Unicode, and separator-containing names.

## PRRT_kwDOTNkCxM6QARrS — sanitizer removes every control character

Finding: A text sanitizer that preserves newline, carriage return, or tab can leave log/path injection channels.

Normative resolution:
- The path/log sanitizer must remove or encode all control characters, including NUL, newline, carriage return, and tab.
- It must not reuse a “printable text” sanitizer whose contract intentionally preserves line formatting when the value is used as a path or security-sensitive log field.
- The resulting representation must remain bounded and unambiguous.

Focused verification before resolving this thread:
- Test every ASCII control range plus newline, carriage return, tab, and NUL in path and log fields.
- Assert no control byte can create a second log line, header, or path component.
- Confirm valid Unicode/path characters remain represented according to the documented policy.

## PRRT_kwDOTNkCxM6QARrU — chat probes are local or handshake-bound

Finding: A chat probe that contacts an arbitrary published origin can leak the local chat capability and violates the local-proxy boundary.

Normative resolution:
- Default probes may contact only localhost/127.0.0.1 (including the documented local transport variants).
- Any non-local endpoint requires an explicit authenticated handshake, such as a matching configHash or equivalent server-issued capability, before a request is sent.
- A published viewer origin must not receive API keys or a probe that can be replayed as a chat capability.

Focused verification before resolving this thread:
- Run probes against localhost, 127.0.0.1, an arbitrary published origin, and a non-local endpoint with and without a valid handshake.
- Assert local approved cases work, non-local unapproved cases are rejected before network access, and invalid/mismatched handshakes fail.
- Inspect request headers/body and logs to ensure API keys and reusable capability material never reach the browser or untrusted origin.

## Scope and review boundary

This file is a design/acceptance contract only. It is not evidence that the implementation or focused checks have already passed. After the relevant implementation and validation evidence exists, each mapped existing thread may be replied to and resolved individually. No Bot review will be triggered again.