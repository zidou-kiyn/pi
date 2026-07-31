# RPC And Print

## When This Applies

Working in `packages/coding-agent/src/modes/rpc/**` or
`packages/coding-agent/src/modes/print-mode.ts`.

## The Local Pattern

### RPC framing is strict LF-only JSONL, not `node:readline`

`jsonl.ts` `attachJsonlLineReader` implements its own buffered line splitter
instead of `node:readline`:

> This intentionally does not use Node readline. Readline splits on
> additional Unicode separators that are valid inside JSON strings and
> therefore does not implement strict JSONL framing.

(`jsonl.ts:17-19`.) `docs/rpc.md` "Framing" restates this as a client
requirement: split records on `\n` only, accept an optional trailing `\r`,
and avoid generic line readers.

### `RpcCommand`/`RpcResponse` are hand-written unions kept in sync with `docs/rpc.md`

`rpc-types.ts` defines `RpcCommand` as a discriminated union of
`{ id?: string; type: "..."; ... }` variants (`prompt`, `steer`, `abort`,
`set_model`, `compact`, `fork`, `get_tree`, ...) and a parallel `RpcResponse`
union of `{ type: "response"; command: "<name>"; success; data? }` shapes.
`rpc-mode.ts` `runRpcMode` builds `success`/`error` response helpers around
this shape and dispatches on `command.type`. There is no schema or codegen
link between `rpc-types.ts` and `docs/rpc.md`; adding a command means
updating the type union, the handler, and the doc by hand.

### Print mode is the single-shot case of the same runtime host

`print-mode.ts` `runPrintMode` takes the same `AgentSessionRuntime` as
interactive/RPC mode, sends `options.initialMessage` and any `options.messages`
through `session.prompt(...)`, then for `mode: "text"` prints only the final
assistant message text, or for `mode: "json"` prints the session header once
followed by every `AgentSessionEvent` as one JSON line via
`session.subscribe`. This is the same event surface documented for
`pi --mode json` in `docs/json.md`.

### Both modes install their own signal handlers around runtime teardown

`print-mode.ts` and `rpc-mode.ts` each register `SIGTERM` (and `SIGHUP` on
non-Windows) handlers that call `killTrackedDetachedChildren()` and then
dispose the runtime before `process.exit`, tracking the handlers in a local
`signalCleanupHandlers` array so they can be removed again in `finally`.
Neither mode relies on a `process.on("exit")` handler for cleanup.

### `RpcClient` is the reference subprocess client, not a cross-language spec

`rpc-client.ts` `RpcClient` spawns the CLI with `--mode rpc`, reuses
`attachJsonlLineReader` for parsing, and exposes one typed method per
command. `docs/rpc.md` tells Node/TypeScript embedders to prefer
`AgentSession` directly over spawning a subprocess, and points non-Node
clients at `rpc-client.ts` only as a worked example of the wire protocol.

## Reference Files

- `packages/coding-agent/src/modes/rpc/rpc-mode.ts`
- `packages/coding-agent/src/modes/rpc/rpc-types.ts`
- `packages/coding-agent/src/modes/rpc/rpc-client.ts`
- `packages/coding-agent/src/modes/rpc/jsonl.ts`
- `packages/coding-agent/src/modes/print-mode.ts`
- `packages/coding-agent/docs/rpc.md`
- `packages/coding-agent/docs/json.md`

## Anti-Patterns

- Parsing RPC stdin with `node:readline` or any line splitter that treats
  `U+2028`/`U+2029` as line breaks — both are valid inside JSON string
  payloads and would corrupt framing.
- Adding a new `RpcCommand` variant without a matching `RpcResponse` variant
  and a `docs/rpc.md` section describing it.
- Reimplementing session replacement (new/fork/switch) inside a mode instead
  of calling the corresponding `AgentSessionRuntime` method — the runtime
  already owns teardown and recreation of cwd-bound services.
