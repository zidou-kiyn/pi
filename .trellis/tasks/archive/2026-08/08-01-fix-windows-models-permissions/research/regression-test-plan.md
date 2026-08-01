# Regression-test plan

## Platform simulation helper

Use a helper that saves `Object.getOwnPropertyDescriptor(process, "platform")`, temporarily defines `process.platform` as `"win32"`, `"linux"`, or `"darwin"`, awaits the test body, and restores the descriptor in `finally`. Existing pi tests use this descriptor-restore pattern. Keep platform-mutating tests non-concurrent within a test file.

For a writable Windows `stat()` result on a POSIX test host, `chmodSync(path, 0o666)` is a useful simulation: Node/libuv reports a normal writable Windows file as `0666` in the mode bit-field. Do not assert exact `0600`/`0400` values in the simulated Windows branch.

## Required Windows regressions

1. **First add** (`test/models-wizard-workflow.test.ts`): run the injected TUI workflow with no existing `models.json` under simulated `win32`. Assert the selected provider is written and the success notification is emitted. Assert content/backup behavior, not an exact numeric mode.
2. **Second add / exact no-op**: run the first-add setup, then set the file to simulated writable mode `0666` and run the same workflow again under `win32`. Assert:
   - the second run reports “already configured”;
   - the API-key prompt is allowed to complete, while `apply` is not called;
   - bytes and `mtimeMs` remain unchanged;
   - no `.preset-bak` is created or replaced by the no-op.

   Before the preflight fix, this test fails with `file permissions are broader than owner-only` before planning the no-op.
3. **Replacement**: create an existing provider, set the file to simulated writable `0666`, and run the workflow under `win32` with a changed URL/key/model selection and both confirmations accepted. Assert the provider object is replaced as a unit, stale fields are gone, the sibling provider survives, and the original bytes are in `.preset-bak`. Do not make backup/target mode equality a Windows assertion.

These should be workflow-level tests because the reported failure occurs in `readModelsDocument()` before the command can plan or apply; `applyProviderPlan()` also re-reads, so the replacement test covers both preflight calls.

## POSIX safety regressions

Retain or make explicit tests for both simulated `linux` and simulated `darwin`:

- An existing `0640`/`0666`-style broad mode still throws the actionable `chmod 600` error before the masked key prompt and before any write.
- Existing owner-only modes continue to pass the preflight and remain unchanged by the atomic writer (the current `0600`/`0400` tests cover this on POSIX hosts).
- Existing symlink, dangling-symlink, backup, and TOCTOU tests remain unchanged.

The broad-mode test currently lives in `test/models-config.test.ts:105-127`; the workflow-level broad-permission/key-prompt test is at `test/models-wizard-workflow.test.ts:445-465`.

## Mode-assertion updates

The following exact numeric mode assertions need a POSIX condition or a Windows-specific writable/read-only assertion, otherwise valid Windows behavior will fail tests:

- `test/json-merge.test.ts:51-56` (new `0600` file);
- `test/json-merge.test.ts:66-76` (existing `0400` and backup);
- `test/json-merge.test.ts:82-90` (legacy `0640` preservation);
- `test/models-wizard-workflow.test.ts:205-229` (successful first add);
- `test/models-config-apply.test.ts:199-219` (symlink target and backup modes).

The POSIX branches must continue to assert exact owner-only modes. Windows branches should assert successful content/transaction behavior and, where useful, the writable bit/read-only state rather than owner/group/other equality.
