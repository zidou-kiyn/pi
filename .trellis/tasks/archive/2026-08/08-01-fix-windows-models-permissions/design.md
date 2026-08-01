# Design: Windows models permission compatibility

## Boundary

The fix belongs in `/home/heixiaohu/pi-preset`. No Pi core API or provider-template change is required.

## Permission contract

`readModelsDocument()` keeps resolving and inspecting the real target path. Its permission policy becomes:

```text
win32          -> do not interpret group/other bits; continue with normal filesystem access checks
linux/darwin   -> reject when mode & 0077 is nonzero
other POSIX    -> retain the current POSIX-style gate
```

The shared writer continues requesting `0600` for a new file and preserving an existing numeric mode. On Windows those values express writable/read-only behavior through Node, not ACL ownership.

## Test seam

Add a narrow platform input to the permission decision rather than mutating unrelated writer behavior. Tests can pass or temporarily simulate `win32`, `linux`, and `darwin` deterministically.

Workflow regressions cover:

1. Windows first add;
2. Windows second identical add as a byte/mtime-preserving no-op;
3. Windows provider replacement with backup and sibling preservation;
4. unchanged Linux/macOS broad-mode rejection.

Exact `0600`/`0400` assertions remain binding only on POSIX branches. Windows assertions focus on successful content and transaction behavior.

## Compatibility and rollback

- Existing POSIX behavior is unchanged.
- Windows stops receiving an invalid `chmod 600` recovery instruction for synthetic `0666` modes.
- Reverting the platform guard restores the previous behavior; no on-disk migration is required.
