# Modes Guidelines

Governs `packages/coding-agent/src/modes/interactive/**`,
`packages/coding-agent/src/modes/rpc/**`,
`packages/coding-agent/src/modes/print-mode.ts`,
`packages/coding-agent/src/cli/`, `src/main.ts`, and `src/cli.ts` — the three
run-mode entry points (interactive TUI, RPC, print/json) and the CLI wiring
that selects between them.

## Pre-Development Checklist

1. Read `.trellis/spec/_shared/index.md` first.
2. Identify which mode you are changing: interactive (`InteractiveMode`),
   RPC (`runRpcMode`), or print/json (`runPrintMode`). All three receive the
   same `AgentSessionRuntime` constructed once in `main.ts`
   (`core/agent-session-runtime.ts`) — never construct a second
   `AgentSessionRuntime`/`AgentSession` inside a mode.
3. If the change affects session replacement (new session, fork, switch,
   `/cd`), check how the other two modes implement
   `runtimeHost.setRebindSession(...)` before adding mode-specific logic —
   `interactive-tui.md` and `rpc-and-print.md` document the shared
   rebind/`bindExtensions`/`subscribe` triad.
4. If the change adds or changes an RPC command or event, update
   `rpc-types.ts`, `rpc-mode.ts`, and `packages/coding-agent/docs/rpc.md` in
   the same change; the doc is hand-maintained, not generated.
5. If the change adds a new interactive keyboard action, add it to
   `core/keybindings.ts` (`KEYBINDINGS`/`AppKeybindings`) and
   `packages/coding-agent/docs/keybindings.md`; never hardcode a literal key
   check (`AGENTS.md` Code Quality rule).
6. Plan verification: `npm run check`; targeted vitest for mode-agnostic
   behavior under `packages/coding-agent/test/`; the tmux workflow from
   `_shared/testing.md` for anything that only shows up in a real terminal.

## Guidelines Index

| File | Covers |
|---|---|
| [Interactive TUI](./interactive-tui.md) | `InteractiveMode` structure, `TuiMainScreen`/`TuiAltScreen`, keybindings, tool render integration |
| [RPC And Print](./rpc-and-print.md) | JSONL framing, `RpcCommand`/`RpcResponse` contract, print/json single-shot mode, shared signal handling |

## Shared Rules

Repo-wide rules (TypeScript/style, testing, `npm run check`, deps/git):
[`.trellis/spec/_shared/index.md`](../../_shared/index.md).

Generic thinking aids (cross-layer, code reuse):
[`.trellis/spec/guides/index.md`](../../guides/index.md).
