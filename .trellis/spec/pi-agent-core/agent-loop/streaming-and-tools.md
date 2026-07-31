# Streaming And Tools

## When This Applies

Any change to the turn loop, emitted event order, the `StreamFn` contract, tool
preparation/execution, or the proxy stream: `packages/agent/src/agent-loop.ts`,
`agent.ts`, `stream-fn.ts`, `proxy.ts`, `types.ts`.

## The Local Pattern

### Failures are values, not exceptions — except in config callbacks

`StreamFn` (`types.ts:28`) must not throw or reject; failures are encoded in the
returned stream and in a final `AssistantMessage` with `stopReason` `"error"`
or `"aborted"`. `streamProxy` (`proxy.ts:116`) implements exactly that — its
`catch` sets `partial.stopReason`, pushes `{ type: "error" }`, then calls
`stream.end()`.

Tool failures are caught at three points and become error tool results via
`createErrorToolResult` (`agent-loop.ts:756`):

- `prepareToolCall` (`agent-loop.ts:600`) — unknown tool, schema validation
  failure, `beforeToolCall` throwing, `beforeToolCall` returning
  `{ block: true }`, or an aborted signal, all returning `kind: "immediate"`.
- `executePreparedToolCall` (`agent-loop.ts:666`) — `tool.execute()` throwing.
- `finalizeExecutedToolCall` (`agent-loop.ts:709`) — `afterToolCall` throwing.

`AgentLoopConfig` callbacks (`convertToLlm`, `transformContext`,
`getSteeringMessages`, `getFollowUpMessages`, `shouldStopAfterTurn`,
`prepareNextTurn`) are **not** wrapped. `agentLoop` calls
`void runAgentLoop(...).then(...)` with no `.catch` (`agent-loop.ts:40-51`), so
a rejection there becomes an unhandled rejection and the returned `EventStream`
never ends. `Agent.runWithLifecycle` (`agent.ts:490`) is the only layer that
catches; `handleRunFailure` (`agent.ts:496`) synthesizes the missing
`message_start` / `message_end` / `turn_end` / `agent_end` quartet
(`test/agent.test.ts` "emits full lifecycle events for thrown run failures").

### Loop shape: outer follow-up loop, inner tool/steering loop

`runLoop` (`agent-loop.ts:155`) is a `while (true)` outer loop (`:170`) around
`while (hasMoreToolCalls || pendingMessages.length > 0)` (`:174`). Per turn:

1. Steering messages, seeded before the first turn (`:167`) and re-polled at
   `:259`.
2. Assistant stream, tool batch, `turn_end`.
3. `prepareNextTurn` (`:232`) — may replace context, model, or thinking level
   for the next request only.
4. `shouldStopAfterTurn` (`:248`) — exits before either queue is polled again.
5. `getFollowUpMessages` (`:263`) — only reached when the inner loop drains.

`test/agent-loop.test.ts` "should stop after the current turn when
shouldStopAfterTurn returns true" pins this: one steering poll, zero follow-up
polls, and the exact twelve-event sequence. An assistant message with
`stopReason` `"error"` or `"aborted"` short-circuits earlier
(`agent-loop.ts:196`): `turn_end` with empty `toolResults`, then `agent_end`,
no queue polled.

### Tool batches: one mode switch, two orderings

`executeToolCalls` (`agent-loop.ts:411`) picks sequential when
`config.toolExecution === "sequential"` **or** any called tool declares
`executionMode: "sequential"` (`:419`) — a single sequential tool forces the
whole batch. `executeToolCallsParallel` (`:489`) prepares calls sequentially,
then resolves them with `Promise.all`, producing two different orders on
purpose: `tool_execution_end` follows completion order, while `toolResult`
messages and `turn_end.toolResults` follow assistant source order. Test "should
emit tool_execution_end in completion order but persist tool results in source
order" asserts `["tool-2", "tool-1"]` against `["tool-1", "tool-2"]`.

Early exit is unanimous only: `shouldTerminateToolBatch` (`agent-loop.ts:582`)
requires every finalized result to set `terminate === true`.

### `stopReason: "length"` never executes tools

`agent-loop.ts:212` routes a length-truncated message to
`failToolCallsFromTruncatedMessage` (`:381`). Streamed tool-call arguments are
finalized by a best-effort salvage JSON parser, so truncated arguments can pass
schema validation. Every call in that message gets an error result asking the
model to re-issue it; the loop continues.

### Streaming assembly mutates the passed-in context

`streamAssistantResponse` (`agent-loop.ts:281`) pushes the partial assistant
message into `context.messages` on `start`, replaces the tail on every delta,
then swaps in `await response.result()` on `done`/`error`. Consequence:
`runAgentLoopContinue` shallow-copies with `{ ...context }`
(`agent-loop.ts:136`), so the **caller's** `messages` array is appended to,
while `runAgentLoop` builds a fresh array (`:106`) and does not. `Agent` avoids
both by snapshotting with `.slice()` in `createContextSnapshot`
(`agent.ts:426`) and rebuilding its transcript from `message_end` events
(`agent.ts:529`). Direct low-level callers must copy the array themselves.

The same function awaits `config.getApiKey(config.model.provider)` immediately
before each `streamFunction` call, falling back to `config.apiKey`
(`agent-loop.ts:305`). It exists for short-lived OAuth tokens that expire
during long tool phases; do not hoist it out of the loop.

### The default stream function is a compatibility fallback

`stream-fn.ts` holds one module-level `defaultStreamFn`. Both runners apply
`streamFn ?? getDefaultStreamFn()` (`agent-loop.ts:116`, `:141`) and the
`Agent` constructor does the same (`agent.ts:216`), under the comment "Older
compiled consumers may omit options or streamFn even though the current API
requires them". The types mark both required; the runtime fallback is
deliberate compatibility, reached by tests through `Reflect.apply` /
`Reflect.construct`. Only `setDefaultStreamFn` is re-exported from
`src/index.ts`; `getDefaultStreamFn` stays internal and throws when unset.

### Event delivery differs by tier

The `EventStream` wrappers push and move on — the low-level streams are
observational and do not let a consumer's async handling gate later producer
phases (`packages/agent/README.md`, "Low-Level API"). `Agent.processEvents`
awaits every listener in registration order (`agent.ts:574`), which is why
assistant `message_end` is a barrier before tool preflight and why
`agent.prompt()` settles only after awaited `agent_end` listeners finish
(`test/agent.test.ts` "should await async subscribers before prompt resolves").
Within one execution, `executePreparedToolCall` (`agent-loop.ts:666`) flips
`acceptingUpdates` to `false` once `execute()` settles, so late `onUpdate`
calls are dropped silently rather than thrown ("should ignore tool updates
after the tool execution settles").

## Reference Files

- `packages/agent/src/agent-loop.ts` — `runLoop`, `streamAssistantResponse`,
  `executeToolCalls*`, `prepareToolCall`, `failToolCallsFromTruncatedMessage`,
  `createToolResultMessage`
- `packages/agent/src/agent.ts` — `Agent`, `PendingMessageQueue`,
  `createLoopConfig`, `runWithLifecycle`, `handleRunFailure`, `processEvents`
- `packages/agent/src/types.ts` — `StreamFn`, `AgentLoopConfig`, `AgentTool`,
  `AgentEvent`, `AgentState` contracts and their binding doc comments
- `packages/agent/src/stream-fn.ts`, `packages/agent/src/proxy.ts`,
  `packages/agent/src/index.ts`, `packages/agent/src/node.ts`
- `packages/ai/src/utils/event-stream.ts` — `EventStream` used by both tiers
- `packages/agent/README.md` — event-sequence tables, steering/follow-up rules
- `packages/agent/test/agent-loop.test.ts` — loop ordering, tool modes,
  truncation guard, `shouldStopAfterTurn`, `prepareNextTurn`
- `packages/agent/test/agent.test.ts` — lifecycle, abort signal, queues, late
  tool updates
- `packages/agent/test/e2e.test.ts` + `packages/ai/src/providers/faux.ts` —
  faux-provider integration
- `packages/agent/test/utils/calculate.ts` — the reference `AgentTool` shape
- `packages/coding-agent/src/core/sdk.ts:294` — the in-repo `new Agent(...)`
  call site; check it when changing `AgentOptions`

`packages/agent/docs/models.md` and `packages/agent/docs/observability.md` are
design notes, not current behavior: `models.md` states it "describes the target
design" for pi-ai, and `observability.md` proposes a `packages/observability`
workspace that does not exist. Do not cite either as the current contract.

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Throwing from `convertToLlm`, `transformContext`, `shouldStopAfterTurn`, `prepareNextTurn`, or the queue getters | `agentLoop` has no `.catch`; the stream never ends and the rejection is unhandled | `agent-loop.ts:40-51`, contracts in `types.ts` |
| Returning an error string as tool `content` instead of throwing | The loop's `isError` flag and error result text come from the thrown error | `agent-loop.ts:666`, `packages/agent/README.md` "Error Handling" |
| Executing tool calls from a message with `stopReason: "length"` | Salvaged arguments can validate while being silently truncated | `agent-loop.ts:381`, "should not execute tool calls from a length-truncated assistant message" |
| Terminating a batch when only one result sets `terminate` | The rule is unanimous, not any-of | `agent-loop.ts:582`, "should continue after parallel tool calls when not all tool results terminate" |
| Emitting `toolResult` messages in completion order in parallel mode | Transcript order must match assistant source order | "should emit tool_execution_end in completion order but persist tool results in source order" |
| Re-validating or re-reading arguments after `beforeToolCall` | The hook mutates the validated object in place and the loop executes it as-is; changing this breaks the documented behavior | "should execute mutated beforeToolCall args without revalidation" |
| Passing a live transcript array straight into `agentLoopContinue` | The array is shared and gets appended to | `agent-loop.ts:136` vs `agent.ts:426` |
| Adding an `AgentEvent` variant without updating `Agent.processEvents` | The reducer silently ignores unknown types and state drifts | `agent.ts:529` |
| Caching the resolved API key across turns | Defeats the expiring-OAuth-token case the hook exists for | `agent-loop.ts:305`, `types.ts` `getApiKey` |
