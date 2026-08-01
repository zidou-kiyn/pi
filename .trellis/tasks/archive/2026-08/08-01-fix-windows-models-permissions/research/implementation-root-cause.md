# Implementation and root cause

## Current flow

- `src/models-config.ts:66-93` runs `statSync(targetPath).mode & 0o777` during every existing-file preflight. It rejects whenever `mode & 0o077` is nonzero, before the API-key prompt.
- `extensions/preset-models-add.ts:94-105` performs that preflight before collecting the key. `src/models-config-apply.ts:36` re-reads the same document after confirmation, so the permission gate is also applied immediately before replacement/write.
- `src/models-config-apply.ts:45-48` passes `newFileMode: 0o600` and `rejectDanglingSymlink: true` to the shared writer.
- `src/json-merge.ts:222-239` preserves an existing numeric mode, creates unique sibling backup/tmp files, applies `fchmodSync`, and renames atomically. The writer is shared with `/preset-sync`.

## Reproduction

A writable Windows file is represented by Node/libuv with a mode whose low bits are effectively `0666`. A POSIX simulation (`chmod 0666` plus a temporary `process.platform = "win32"`) currently produces:

```text
simulated Windows writable stat mode: 666
readModelsDocument: refusing to read .../models.json: file permissions are broader than owner-only; run chmod 600 .../models.json
```

The full workflow reproduces the report: the first add creates the provider successfully; after the file is made `0666` to model the Windows `stat()` result, the second identical invocation stops at `cannot prepare models.json` with the owner-only error. It never reaches the existing-provider no-op decision. Replacement is blocked at the same preflight.

## Narrowest fix

Guard only the POSIX owner/group/other check:

```ts
if (process.platform !== "win32" && (mode & 0o077) !== 0) {
  // existing actionable chmod 600 error
}
```

Do not change the shared atomic writer or disable its backup/tmp/rename sequence. The failure is the preflight interpreting Windows' synthetic POSIX mode, not a failed write. On Windows the numeric mode cannot enforce owner-only access; Windows ACLs are the actual access-control mechanism. Actual read/write/rename failures should still propagate from the normal filesystem operation.

Do not broaden the exception by accepting `0666` based on the numeric value alone: that would weaken the Linux/macOS policy. The platform check must be explicit.

## Atomic-writer implications

`newFileMode: 0o600` remains the correct POSIX request. On Windows, Node treats the mode as a writable/read-only signal, not as separate owner/group/other permissions. A writable target may therefore be reported as `0666` after the writer's `openSync`/`fchmodSync`; this is expected and should not be used as a second-run blocker. Replacement still uses the existing mode for backup and temporary files, and the atomic transaction remains unchanged.

The README's exact `0600` wording should be qualified for Windows if product documentation is updated: exact owner-only mode applies to POSIX filesystems; Windows access is governed by ACLs and Node's numeric mode is only a writable/read-only approximation.

## Baseline

The current standalone model/config/writer/workflow test set passes: 43 tests, 0 failures. The worktree was clean before research; no product files were changed.
