# PRD: Fix Windows models permission check

## Goal

Allow `/preset-models-add` to read and update an existing writable `models.json` on Windows without weakening the owner-only permission policy on POSIX platforms.

## User Outcome

A Windows user can run the models wizard repeatedly—add, exact no-op, or replace—without receiving a false `chmod 600` error after the first successful write.

## Background and Confirmed Facts

- The first Windows add succeeds, but the second run fails in `readModelsDocument()` before the API-key prompt.
- The current preflight rejects an existing file when `statSync(path).mode & 0o077` is nonzero.
- Node/libuv represents a normal writable Windows file with synthetic mode bits equivalent to `0666` because Windows ACLs do not implement POSIX owner/group/other permission classes.
- Node's Windows `chmod` support only manipulates the writable/read-only attribute. `chmod 600` cannot establish or prove a POSIX owner-only policy on Windows.
- The existing atomic writer, backup, symlink, redaction, and provider-upsert behavior is not the cause of the failure.

## Requirements

- **R1 Windows compatibility:** skip the POSIX group/other permission-bit gate when the runtime platform is `win32`.
- **R2 POSIX security:** Linux and macOS must continue rejecting existing files with any group/other permission bits and retain the actionable `chmod 600` instruction.
- **R3 Repeated workflow:** Windows first add, second exact no-op, and replacement of an existing provider must all reach their normal workflow outcomes.
- **R4 Transaction safety:** do not weaken backup, atomic rename, symlink, TOCTOU, redaction, or provider-as-a-unit replacement behavior.
- **R5 Platform-correct diagnostics:** Windows must not receive a POSIX `chmod 600` error based only on Node's synthetic mode bits.
- **R6 Documentation:** qualify the README's exact `0600` guarantee as POSIX-specific and explain that Windows access is governed by ACLs.
- **R7 Test portability:** tests must simulate Windows explicitly and must not assert exact POSIX numeric modes in Windows branches.

## Acceptance Criteria

- **AC1** Under simulated `win32`, an existing writable file represented as mode `0666` passes `readModelsDocument()`.
- **AC2** Under simulated `win32`, running the same provider configuration a second time reports already configured, leaves bytes and mtime unchanged, and creates no backup.
- **AC3** Under simulated `win32`, replacing an existing provider succeeds after both confirmations, preserves sibling providers, removes stale provider fields, and writes the original bytes to `.preset-bak`.
- **AC4** Under simulated `linux` and `darwin`, modes such as `0640` or `0666` still fail before credential entry with the existing `chmod 600` guidance.
- **AC5** Existing model wizard, JSON writer, skills sync, hot-reload, and secret-scan regressions continue to pass.
- **AC6** The public README accurately distinguishes POSIX modes from Windows ACL behavior.

## Out of Scope

- Inspecting, rewriting, or hardening Windows ACLs.
- Adding a Windows ACL-management dependency or invoking `icacls`.
- Changing the models wizard's credential storage design.
- Changing Linux/macOS permission requirements.
- Altering provider templates or model selection behavior.

## Key Decision

- **D1 Platform-specific gate:** use an explicit `win32` branch rather than accepting mode `0666` globally. Numeric owner/group/other bits remain authoritative on POSIX and non-authoritative on Windows.
