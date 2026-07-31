# Core Guidelines

Governs `packages/coding-agent/src/core/**`, `packages/coding-agent/src/config.ts`,
`packages/coding-agent/src/migrations.ts`, and `packages/coding-agent/src/utils/`
— session lifecycle, settings/config resolution, the built-in tools, extension
loading/running, and compaction.

## Pre-Development Checklist

1. Read `.trellis/spec/_shared/index.md` first; this layer never restates
   TypeScript style, testing, or `npm run check` rules.
2. For session/settings/config changes: read `session-and-config.md`, then
   `packages/coding-agent/src/core/agent-session.ts` (event surface),
   `agent-session-services.ts` (services vs. session split), and
   `settings-manager.ts` (global/project layering) before editing.
3. For a new or changed built-in tool: read `tools.md`, then the sibling tool
   file under `packages/coding-agent/src/core/tools/` closest to what you are
   changing (`edit.ts` is the most complete reference implementation).
4. For extension-facing changes (events, `ExtensionAPI`, tool registration):
   read `../extensions/extension-api.md` — the extension loader/runner also
   live under `core/` (`core/extensions/`) but are documented in that layer.
5. `AgentSession` (`agent-session.ts`) is ~3300 lines and touched by almost
   every feature. Grep for the exact event or method name first
   (`guides/code-reuse-thinking-guide.md`) instead of assuming where logic
   lives.
6. Any new persisted field (`Settings`, `SessionEntry`, `CompactionEntry`)
   crosses the session-file/RPC/TUI boundary — read
   `guides/cross-layer-thinking-guide.md` first.
7. Plan verification: `npm run check`, plus the closest existing test file
   under `packages/coding-agent/test/` for the area touched (see
   `testing.md` for the coding-agent suite contract).

## Guidelines Index

| File | Covers |
|---|---|
| [Session And Config](./session-and-config.md) | `AgentSession` event model, services/session split, settings layering, session-file persistence, startup migrations |
| [Tools](./tools.md) | Built-in tool (`read`/`bash`/`edit`/`write`/`grep`/`find`/`ls`) structure, pluggable operations, render lifecycle |

## Shared Rules

Repo-wide rules (TypeScript/style, testing, `npm run check`, deps/git):
[`.trellis/spec/_shared/index.md`](../../_shared/index.md).

Generic thinking aids (cross-layer, code reuse):
[`.trellis/spec/guides/index.md`](../../guides/index.md).
