# Server Guidelines

Covers the experimental supervisor daemon in `packages/server/src/`: the Unix-socket
IPC layer (`packages/server/src/ipc/protocol.ts`, `client.ts`, `server.ts`), the
instance supervisor (`supervisor.ts`, `rpc-process.ts`), the request handler
(`handler.ts`), the daemon entry points (`serve.ts`, `cli.ts`), and the local
persistence and Radius presence helpers (`config.ts`, `storage.ts`, `radius.ts`,
`types.ts`).

`packages/server/README.md` marks this package experimental: its CLI, APIs, and
behavior are not stable, and no other workspace depends on it. That is licence to
change shapes freely, not licence to skip the conventions below.

## Pre-Development Checklist

1. Read `../../_shared/typescript-and-style.md` and `../../_shared/testing.md`
   first.
2. Decide which of the three surfaces you are touching, because each has a
   different blast radius:
   - wire protocol (`src/ipc/protocol.ts`) — changing it breaks `src/cli.ts`,
     `src/handler.ts`, and `src/ipc/server.ts` simultaneously;
   - supervisor state (`src/supervisor.ts`, `src/storage.ts`) — changing it
     changes what is written to `instances.json`;
   - child-process bridge (`src/rpc-process.ts`) — it speaks the coding-agent RPC
     protocol defined in `packages/coding-agent/src/modes/rpc/rpc-types.ts`, which
     this package does not own.
3. If you add or rename a request type, walk the full fan-out listed in
   `ipc-and-supervisor.md` "Adding a request type". Missing one site compiles in
   some cases because `parseRequestLine` is an unchecked cast.
4. If the change crosses into `@earendil-works/pi-coding-agent` types
   (`RpcCommand`, `RpcResponse`, `AgentSessionEvent`, `RpcExtensionUIRequest`),
   read `../../guides/cross-layer-thinking-guide.md` before editing: the same
   payload is re-serialized three times (child stdout, IPC socket, CLI stdout).
5. Never construct filesystem paths from `homedir()` in new code — use the
   accessors in `src/config.ts` so `PI_SERVER_DIR` and `PI_CONFIG_DIR` keep
   working.
6. Verify with `npm run check` from the repo root. This package has **no test
   directory and no `test` script**, so `tsgo --noEmit` and `biome` are the only
   automated gates; exercise behavioral changes by hand with
   `node packages/server/src/cli.ts serve` in one shell and `... cli.ts list` /
   `spawn` / `rpc-stream` in another.

## Guidelines Index

| File | Covers |
|---|---|
| [IPC And Supervisor](./ipc-and-supervisor.md) | JSONL socket framing, one-shot vs streaming connections, request-type fan-out, supervisor lifecycle and write-through persistence, RPC child-process bridge, Radius presence retry policy |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
