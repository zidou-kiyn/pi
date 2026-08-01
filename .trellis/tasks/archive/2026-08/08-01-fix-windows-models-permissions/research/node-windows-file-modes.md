# Node.js Windows file-mode behavior

Authoritative Node.js v24.16.0 references:

- [`fs.Stats.mode`](https://nodejs.org/download/release/v24.16.0/docs/api/fs.html#statsmode) defines `stats.mode` only as “a bit-field describing the file type and mode.”
- [`File modes`](https://nodejs.org/download/release/v24.16.0/docs/api/fs.html#file-modes), under `fs.chmod()`/`fs.chmodSync()`, documents the Windows caveat: only the write permission can be changed, and the distinction among group, owner, and others is not implemented.
- [`fs.open()` mode](https://nodejs.org/download/release/v24.16.0/docs/api/fs.html#fsopenpath-flags-mode-callback) says the mode applies when the file is created; on Windows only write permission can be manipulated.
- Node's own [`test-fs-chmod.js`](https://github.com/nodejs/node/blob/v24.16.0/test/parallel/test-fs-chmod.js#L64-L117) explicitly branches for Windows and checks permission-bit intersection rather than exact POSIX modes.

The exact Node v24.16.0 Windows implementation is in its vendored libuv source, [`deps/uv/src/win/fs.c`](https://github.com/nodejs/node/blob/v24.16.0/deps/uv/src/win/fs.c#L1928-L1960):

- `FILE_ATTRIBUTE_READONLY` maps to read bits for owner/group/others (`0444`).
- A file without that attribute maps to read and write bits for all three classes (`0666`).
- The source comments explain that Windows uses ACLs and that the synthetic `st_mode` does not have POSIX security semantics.

Therefore `statSync(path).mode & 0o777` cannot answer the models command's POSIX question (“are group/other permissions absent?”) on Windows. A normal writable `models.json` is expected to look broad to the current preflight even after `fchmodSync(..., 0o600)`, because Windows only tracks the writable/read-only attribute through this API. The current `mode & 0o077` rejection is consequently a false positive, not evidence of a group-readable credential file.

The narrow platform-specific behavior is:

- Linux/macOS: keep rejecting any existing mode with group/other bits; keep preserving exact owner-only modes in the writer.
- Windows: skip this numeric owner-only gate and rely on the filesystem's Windows ACL/access checks. Keep the atomic backup/tmp/rename and normal error handling.
