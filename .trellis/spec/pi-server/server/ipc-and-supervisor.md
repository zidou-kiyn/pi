# IPC And Supervisor

## When This Applies

Any change to the socket protocol, the daemon lifecycle, instance bookkeeping,
the child RPC process bridge, or Radius presence — that is, any file under
`packages/server/src/` or `packages/server/src/ipc/`. Paths below are relative to
`packages/server/src/`.

## The Local Pattern

### One message per line, one type union per direction

`encodeMessage()` (`ipc/protocol.ts`) is the only serializer: it appends `\n` and
nothing else. Every reader re-implements the same newline framer —
`ipc/server.ts` inside `createServer`, `ipc/client.ts:sendIpcRequest`,
`rpc-process.ts:RpcProcessInstance.attachListeners`, `cli.ts:rpcStream`: append
to a `buffer` string, split at `\n`, `trim()`, skip empty lines.

Requests and responses are keyed unions. `RequestMap` and `ResponseMap`
(`ipc/protocol.ts`) pair each request `type` with its response `type`, and
`ResponseFor<T>` derives the response from the request at the type level; new
messages go into both maps. `parseRequestLine` / `parseResponseLine` are
`JSON.parse` plus a cast — no runtime validation of inbound frames, which is
known debt given the trust boundary is the local user owning `getServerDir()`.
Do not add casts elsewhere to "match the style"; narrow through the
`switch (request.type)` in `handler.ts:handleIpcRequest` instead.

### Two connection shapes on the same socket

`startIpcServer` (`ipc/server.ts`) serves one-shot requests by responding and
closing: `socket.end(encodeMessage(response))`. `rpc_stream` is the exception —
it upgrades the connection: `handler(request)` is awaited and a non-`rpc_ready`
or `ok: false` result ends the socket; `socket.removeAllListeners("data")`
detaches the one-shot reader; `handler.openRpcStream(...)` returns the bridge (an
`undefined` return closes the socket with `Unknown instance: <id>`); a second
`data` handler drains **all** frames in a loop and serializes them through
`rpcRequestQueue`, a promise chain, so writes cannot interleave RPCs; and
`socket.once("close", () => rpcStream.close())` unsubscribes. Errors inside the
stream are written back as `ErrorResponse` frames, never by closing.

`removeStaleSocketIfNeeded` + `isSocketLive` run before `listen`: they connect to
an existing socket path first, and a successful connect throws
`server is already running: <path>`. `ECONNREFUSED`, `ENOENT`, `EPIPE`, and
`ECONNRESET` mean stale and the file is unlinked; any other errno rejects. A bare
`unlinkSync` would let a second daemon steal a live socket.

### Supervisor state is write-through and cloned on the way out

`ServerSupervisor` (`supervisor.ts:63`) keeps `liveInstances: Map<string,
LiveInstance>` in memory, where `LiveInstance` bundles the persisted
`InstanceRecord`, non-serializable `resources` (child process, radius id), and
event `subscribers`. Every mutation goes through `setStatus` or `updateRecord`,
which stamp `lastSeenAt` and immediately call `upsertInstance` (`storage.ts`) to
rewrite `instances.json`; `live.record` is never mutated in place, it is replaced
with a spread copy. Every record leaving the supervisor passes through
`cloneInstance`, so callers such as `handler.ts:toInstanceSummary` cannot mutate
live state. `storage.ts` is the only module that touches `instances.json` and
`machine.json`, reloading the full array on every call.

### Session metadata is refreshed from an explicit allowlist

`SESSION_METADATA_COMMANDS` (`supervisor.ts:41`) lists the six RPC command types
that can change persisted identity (`new_session`, `switch_session`, `fork`,
`clone`, `set_session_name`, `prompt`). Only those trigger `syncInstanceRecord`,
which issues a follow-up `get_state` and narrows the reply with
`isGetStateSuccess`. Its comment gives the reason: most RPCs mutate transient
runtime state only, so a `get_state` after every command is wasted IO. Extend the
set deliberately; do not make the refresh unconditional.

### Failure paths are idempotent and identity-checked

`failSpawn` marks `error`, runs `cleanupAcquiredResources` in a `try`, then in
`finally` marks `stopped` and drops the map entry before rethrowing;
`stopInstance` uses the same split, so the map entry and the `instances.json`
row disappear even if cleanup throws. `handleUnexpectedRpcExit` first checks
`this.liveInstances.get(live.record.id) !== live` and returns if the entry was
replaced, then returns early when the status is already `stopping`/`stopped`;
Radius disconnect failures there are logged, not thrown.

### The child bridge multiplexes three message kinds

`RpcProcessInstance` (`rpc-process.ts:25`) spawns the coding agent as a child:
`pi --mode rpc` next to `process.execPath` when `isBunBinary` (`config.ts:16`),
otherwise `process.execPath` with
`require.resolve("@earendil-works/pi-coding-agent/rpc-entry")` — a real export of
`packages/coding-agent/package.json`; keep the two in sync. `handleLine` routes
by `parsed.type`: `response` resolves the pending promise registered under its
`id`, `extension_ui_request` goes to the single `uiRequestHandler`, and the
`default` branch is broadcast to `eventListeners` as an `AgentSessionEvent`.
`send()` always assigns an id (`server_${n}_${randomUUID()}`) before writing.
stderr accumulates into `stderrBuffer` and is appended to every error message —
the only diagnostic available when a child dies during spawn.

### Radius presence is optional and self-rescheduling

Every network method in `radius.ts` returns early unless `isRadiusEnabled()`
(stored `radius` OAuth credential or `RADIUS_API_KEY`). Heartbeats use
self-rescheduling `setTimeout`, never `setInterval`. HTTP 404 means "server
forgot us": after `NOT_FOUND_RETRY_THRESHOLD` (3) consecutive 404s the machine or
Pi re-registers; any other failure uses `computeBackoffDelayMs` (exponential from
1s, capped at 30s, plus jitter). `RadiusPresence` reaches back into the
supervisor through `RadiusPresenceCoordinator`, wired at import time by
`radiusPresence.setCoordinator({...})` at the bottom of `supervisor.ts`. Both
`supervisor` and `radiusPresence` are module-level singletons; another
import-time side effect would make initialization order load-bearing.

### Adding a request type

The fan-out is manual, in this order: (1) `ipc/protocol.ts` — request interface,
`RequestMap`, response interface, `ResponseMap`; (2) `handler.ts` — one more
`handleIpcRequest` overload plus a `case` (the switch is exhaustive over
`ServerRequest`, so `tsgo` catches a missing case); (3) `ipc/server.ts` — a
matching call signature on `IpcRequestHandler`; (4) `cli.ts` — the argv branch
and the `printHelp()` usage block, plus `packages/server/README.md` when the CLI
surface changes.

`IpcRequestHandler.openRpcStream` (`ipc/server.ts:33`) and
`handler.ts:openRpcStream` declare the same callback with different parameter
types (`RpcRequest["command"] | { type: "extension_ui_response" }` vs
`RpcCommand | RpcExtensionUIResponse`). They are structurally compatible today
and must be edited together; this duplication is debt, not a pattern to copy.

## Reference Files

- `packages/server/src/ipc/protocol.ts` — `RequestMap`, `ResponseMap`,
  `ResponseFor`, `encodeMessage`; `packages/server/src/ipc/server.ts` —
  `startIpcServer`, `isSocketLive`; `packages/server/src/ipc/client.ts` —
  `sendIpcRequest` settle/cleanup
- `packages/server/src/handler.ts` — `handleIpcRequest` overloads + switch,
  `openRpcStream`, `toInstanceSummary`
- `packages/server/src/supervisor.ts` — `ServerSupervisor`,
  `SESSION_METADATA_COMMANDS`, `cloneInstance`, `handleUnexpectedRpcExit`
- `packages/server/src/rpc-process.ts` — `RpcProcessInstance`, `getSpawnCommand`,
  `handleLine`; `packages/server/src/serve.ts` — shutdown deduplication via
  `shutdownPromise`, `uncaughtException`/`unhandledRejection` wiring
- `packages/server/src/storage.ts`, `packages/server/src/config.ts`,
  `packages/server/src/radius.ts`, `packages/server/src/cli.ts`
- `packages/coding-agent/src/modes/rpc/rpc-types.ts` — the `RpcCommand` /
  `RpcResponse` union this package bridges

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Reading or writing `instances.json` outside `src/storage.ts` | Bypasses `ensureServerDir` and the load-modify-save cycle; two writers clobber each other | `storage.ts` is the sole owner; `supervisor.ts` never touches `fs` |
| Returning `live.record` directly instead of `cloneInstance(...)` | Callers would mutate supervisor state; every accessor already clones | `supervisor.ts:235`, `240`, `257`, `261` |
| `setInterval` for heartbeats | Overlapping in-flight requests when the endpoint is slow | `radius.ts:scheduleMachineHeartbeat`, `schedulePiHeartbeat` |
| Calling `get_state` after every RPC | Wasted IO for commands that only change transient state | Comment above `SESSION_METADATA_COMMANDS`, `supervisor.ts:41` |
| Building paths with `join(homedir(), ".pi", ...)` in new code | Breaks `PI_SERVER_DIR` / `PI_CONFIG_DIR` overrides | `config.ts:getServerDir` is the single source |
| Deep-importing coding-agent internals | This package imports only `@earendil-works/pi-coding-agent` and its published `/rpc-entry` subpath | `packages/coding-agent/package.json` `exports` |
| Adding a request type without updating all four sites | `parseRequestLine` casts, so a missing branch fails at runtime, not at build | See "Adding a request type" |
| Throwing out of a Radius call when the integration is disabled | Every entry point returns early on `!isRadiusEnabled()`; `serve.ts` logs the disabled state and keeps running | `radius.ts:start`, `registerPi`, `disconnectPi` |

Known debt, to be recorded as debt rather than restated as planned work: this
package has no `test/` directory and no `test` script in
`packages/server/package.json`, so `./test.sh` gives it zero coverage;
`config.ts:getAuthPath` and `storage.ts:deleteMachine` are exported through
`src/index.ts` with no caller in the repo; inbound frames are cast, not validated.
