# Harness Guidelines

Covers the harness surface of `packages/agent`: `src/harness/agent-harness.ts`,
`src/harness/types.ts`, `src/harness/session/`, `src/harness/compaction/`,
`src/harness/tools/`, `src/harness/utils/`, `src/harness/env/nodejs.ts`,
`src/harness/messages.ts`, `src/harness/skills.ts`,
`src/harness/prompt-templates.ts`, `src/harness/system-prompt.ts`.

## Pre-Development Checklist

1. Read `../../_shared/typescript-and-style.md` and `../../_shared/testing.md`
   first.
2. Identify which contract layer you are in. `packages/agent/docs/agent-harness.md`
   fixes the split: capabilities and helpers (`ExecutionEnv`, filesystem/shell,
   resource loaders, compaction helpers) return `Result<TValue, TError>` and must
   never throw; `Session` and `AgentHarness` throw typed errors instead. Do not
   mix the two in one function.
3. Treat `packages/agent/docs/agent-harness.md` as the description of implemented
   behavior. `packages/agent/docs/harness.md`, `harness-v2.md`, and `hooks.md` are
   unimplemented designs (durable runs, lanes/refs, an `AgentHarnessHooks` class).
   Never cite them as current behavior; the shipped hook API is
   `AgentHarness.subscribe()` / `AgentHarness.on()` in
   `packages/agent/src/harness/agent-harness.ts`.
4. If you add or change a `SessionTreeEntry` variant, walk the whole chain before
   editing: entry union in `src/harness/types.ts`, `Session.append*`, all three
   `SessionStorage` implementations, context projection, compaction cut-point
   switch, and `convertToLlm`. See `session-and-storage.md`.
5. Keep Node-only imports out of everything reachable from
   `packages/agent/src/index.ts`. `node:*` imports live in
   `src/harness/env/nodejs.ts`, which is re-exported only from
   `packages/agent/src/node.ts`. `scripts/check-browser-smoke.mjs` bundles the
   package for `platform: "browser"` in `npm run check`.
6. Export any new public symbol from `packages/agent/src/index.ts`; that file is
   the package's only public surface besides `src/node.ts`.
7. Run the harness suite from `packages/agent`: `npm run test:harness`
   (`vitest --run --config vitest.harness.config.ts`, which includes
   `test/harness/**/*.test.ts`). Use `npm run coverage:harness` when you add a
   subsystem. Never run the bare vitest suite — see `../../_shared/testing.md`.
8. Use the `pi-ai` faux provider for anything that reaches a model; see
   `packages/agent/test/harness/agent-harness-stream.test.ts` for the
   `fauxProvider` / `models.setProvider` setup used across harness tests.
9. Run `npm run check` from the repo root after any `.ts` edit.

## Guidelines Index

| File | Covers |
|---|---|
| [Session And Storage](./session-and-storage.md) | Entry log and leaf semantics, the three `SessionStorage` backends, context build pipeline, repos and forking, pending session writes |
| [Tools And Compaction](./tools-and-compaction.md) | `AgentHarnessTool` factories, `ExecutionEnv` boundary, mutation queue, truncation limits, compaction and branch summarization, skills and prompt templates |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
