# Agent Loop Guidelines

Covers the provider-agnostic loop primitives of `packages/agent`:
`packages/agent/src/agent-loop.ts`, `agent.ts`, `stream-fn.ts`, `proxy.ts`,
`types.ts`, and the barrels `index.ts` / `node.ts`. Everything under
`packages/agent/src/harness/**` belongs to the `harness` layer of this package,
not here.

## Pre-Development Checklist

1. Read `../../_shared/typescript-and-style.md` and `../../_shared/testing.md`
   first.
2. Decide which of the three tiers you are changing, because they must stay
   consistent:
   - `agentLoop()` / `agentLoopContinue()` (`agent-loop.ts:31`, `:64`) —
     `EventStream` wrappers.
   - `runAgentLoop()` / `runAgentLoopContinue()` (`agent-loop.ts:95`, `:120`) —
     the same run driven by an `AgentEventSink` (`agent-loop.ts:25`).
   - `Agent` (`agent.ts:171`) — stateful wrapper that calls the runners with
     `(event) => this.processEvents(event)` (`agent.ts:407`, `:419`).
3. If you add or change an `AgentEvent` variant (`types.ts` `AgentEvent`),
   update the reducer in `Agent.processEvents` (`agent.ts:529`) and the event
   tables in `packages/agent/README.md`. The union crosses into
   pi-coding-agent, so read `../../guides/cross-layer-thinking-guide.md` before
   changing an existing variant.
4. If you add an `AgentLoopConfig` option (`types.ts`), thread it through
   `AgentOptions` (`agent.ts:97`) and `Agent.createLoopConfig`
   (`agent.ts:434`); otherwise it only works for low-level callers.
5. Keep the "must not throw or reject" doc comments on `AgentLoopConfig`
   callbacks accurate — the loop depends on them, see
   `streaming-and-tools.md`.
6. Any change to emitted event order needs an assertion in
   `packages/agent/test/agent-loop.test.ts`; several tests pin the exact
   `events.map((event) => event.type)` sequence.
7. Run the affected tests from `packages/agent`:
   `node ../../node_modules/vitest/dist/cli.js --run test/agent-loop.test.ts test/agent.test.ts test/e2e.test.ts`.
   Harness coverage runs under a separate config
   (`packages/agent/vitest.harness.config.ts`) and does not cover this layer.
8. Run `npm run check` from the repo root after any `.ts` edit.

## Guidelines Index

| File | Covers |
|---|---|
| [Streaming And Tools](./streaming-and-tools.md) | Loop shape, event ordering, `StreamFn` contract, tool preparation/execution modes, error-as-value handling, context mutation, default stream function |

## Shared Rules

- Repo-wide TypeScript, testing, and check conventions:
  [`../../_shared/index.md`](../../_shared/index.md)
- Generic thinking guides (cross-layer, code reuse):
  [`../../guides/index.md`](../../guides/index.md)
