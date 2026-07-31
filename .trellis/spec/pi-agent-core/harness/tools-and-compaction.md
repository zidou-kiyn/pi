# Tools And Compaction

## When This Applies

Any change under `packages/agent/src/harness/tools/`, `utils/`, `compaction/`,
to `packages/agent/src/harness/skills.ts`, `prompt-templates.ts`,
`system-prompt.ts`, or to the `compact()` / `navigateTree()` paths of
`packages/agent/src/harness/agent-harness.ts`.

## The Local Pattern

### Tools are factories over an injected environment

Every built-in tool is a `createXTool()` factory returning an
`AgentHarnessTool<TContext, TSchema, TDetails>` (`types.ts:99`): a typebox
`parameters` schema plus `execute(toolCallId, params, signal, onUpdate,
context)`, whose fifth argument is the application context, constrained
structurally to `ExecutionToolContext` (`tools/tool-context.ts`) carrying only
`env: ExecutionEnv`, so applications may pass a wider object.
`test/harness/tool-context.types.ts` pins this with a `@ts-expect-error`: a
context-requiring tool without `toolContext` must not compile.

All filesystem and process work goes through `env`, never `node:*`. That keeps
`packages/agent/src/index.ts` browser-bundleable
(`scripts/check-browser-smoke.mjs`); the only Node implementation is
`src/harness/env/nodejs.ts`, exported solely from `src/node.ts`. Even
`utils/truncate.ts:51` avoids a hard `Buffer` dependency by probing
`globalThis.Buffer`. `ExecutionEnv` methods return `Result`; tools convert at
their own boundary with `getOrThrow()` (`types.ts:37`), or by inspecting
`result.ok` when they want a tool-specific message (`editAccessError` in
`tools/edit.ts`).

### Concurrent mutations are serialized per canonical path

`write` and `edit` wrap the whole read-modify-write in
`withFileMutationQueue(env, absolutePath, fn)`
(`tools/file-mutation-queue.ts:29`). The key is the canonical path
(`file-mutation-queue.ts:20`), falling back to the absolute path on
`not_found` / `not_supported`, and state lives in a
`WeakMap<ExecutionEnv, MutationQueueState>` so two environments never contend.
`test/harness/tools.test.ts` proves the cases that matter: the queue stays
locked until an aborted write settles, until an aborted edit write settles, and
two edits through a symlink and its target serialize. Path resolution is also
shared: `resolveToolPath` (`tools/path-utils.ts:12`) strips a leading `@` and
normalizes exotic Unicode spaces; `resolveReadToolPath` (`path-utils.ts:16`)
also probes NFD, typographic-apostrophe, and narrow-no-break-space variants.

### Output limits live in one module

`utils/truncate.ts` owns `DEFAULT_MAX_LINES` (2000) and `DEFAULT_MAX_BYTES`
(50KB). `read` uses `truncateHead` (`truncate.ts:132`); `bash` uses
`truncateTail` (`truncate.ts:222`) via `executeShellWithCapture`
(`utils/shell-output.ts`), which spills full output to a temp file through
`env.appendFile`. Tool descriptions interpolate the constants instead of
repeating the numbers (`tools/read.ts`, `tools/bash.ts`). `tools/edit.ts`
`prepareArguments` heals a JSON-string `edits` field and the legacy flat
`oldText`/`newText` shape and preserves BOM and CRLF; `tools/bash.ts` throttles
`onUpdate` through `BASH_UPDATE_THROTTLE_MS` (100 ms).

### Compaction: prepare (pure) → hook → generate → persist

`AgentHarness.compact()` (`agent-harness.ts:736`) runs a fixed sequence:
`session.getBranch()` → `prepareCompaction()` (`compaction/compaction.ts:640`,
pure, returns `Result<CompactionPreparation | undefined, CompactionError>`) →
the `session_before_compact` hook, which may cancel or supply a ready
`CompactResult` → `compact()` (`compaction.ts:733`) → `appendCompaction` → the
`session_compact` event. Keep new logic on the pure side;
`test/harness/compaction.test.ts` exercises preparation with no provider.

`findValidCutPoints` (`compaction.ts:328`) switches over every entry type and
message role explicitly and never offers a cut before a `toolResult`, so an
assistant tool call is never separated from its results. When the cut lands
inside a turn, `findCutPoint` (`compaction.ts:396`) reports `isSplitTurn` and
`compact()` produces two summaries joined by a
`**Turn Context (split turn):**` separator, merging usages via `combineUsage`.

Summarization requests are isolated: `completeSimpleWithRetries`
(`compaction.ts:118`) forces `cacheRetention: "none"` and a fresh
`sessionId: uuidv7()` so a throwaway summary never pollutes the run's provider
cache affinity. Retries surface as `retry_scheduled` / `retry_attempt_start` /
`retry_finished`, covered by the "summarization retries" block in
`test/harness/agent-harness.test.ts`. `estimateContextTokens`
(`compaction.ts:232`) prefers the real `Usage` of the last non-aborted,
non-error assistant message and estimates only the messages after it; the
fallback is characters / 4, an image counting as `ESTIMATED_IMAGE_CHARS` = 4800
(`compaction.ts:268`).

### Branch summarization mirrors compaction with a different selection

`collectEntriesForBranchSummary` (`compaction/branch-summarization.ts:71`) walks
from the old leaf to the deepest common ancestor with the target, then reverses.
`prepareBranchEntries` (`branch-summarization.ts:127`) fills backwards until the
token budget (`model.contextWindow || 128000` minus `reserveTokens`, default
16384) is exhausted. Both paths share `serializeConversation`,
`computeFileLists`, and `formatFileOperations` from `compaction/utils.ts`.
File-op tracking there is name-coupled: `extractFileOpsFromMessage`
(`compaction/utils.ts:24`) matches the literal tool names `read`, `write`,
`edit` and the literal argument `path`, so renaming a built-in tool or its
`path` parameter silently empties the `<read-files>` / `<modified-files>` blocks
in summaries. That coupling is current behavior and has no guard test; update
this function in the same change as any rename.

### Resource loaders report diagnostics instead of throwing

`loadSkills` (`skills.ts:49`) returns `{ skills, diagnostics }` and
`loadPromptTemplates` (`prompt-templates.ts:30`) returns
`{ promptTemplates, diagnostics }`; neither throws. A missing path is skipped
silently; every other `ExecutionEnv` failure becomes a `type: "warning"`
diagnostic with a stable code. Both have a `loadSourced*` variant attaching an
application-defined `source` the agent package never interprets.

Skill validation (`skills.ts:281`): the name defaults to the parent directory
name, must equal it, must match `^[a-z0-9-]+$`, and is capped at
`MAX_NAME_LENGTH` (64); the description is required and capped at
`MAX_DESCRIPTION_LENGTH` (1024). A skill without a description is dropped, not
warned about. Traversal honors `.gitignore`, `.ignore`, `.fdignore`
(`IGNORE_FILE_NAMES`, `skills.ts:7`) and stops descending once a directory
yields a `SKILL.md`. Known debt: `parseFrontmatter`, `resolveKind`, and
`basenameEnvPath` are verbatim copies in both `skills.ts` and
`prompt-templates.ts`; change both when changing either.

Formatting is separate from loading: `formatSkillInvocation` (`skills.ts:38`),
`formatSkillsForSystemPrompt` (`system-prompt.ts:3`, skips
`disableModelInvocation` skills), and `substituteArgs`
(`prompt-templates.ts:249`: `$1`, `$@`, `$ARGUMENTS`, `${@:N}`, `${@:N:L}`) are
pinned by `test/harness/resource-formatting.test.ts` and `system-prompt.test.ts`.

## Reference Files

- `packages/agent/src/harness/tools/` — `bash.ts`, `edit.ts`, `edit-diff.ts`,
  `read.ts`, `write.ts`, `image.ts`, `path-utils.ts`, `file-mutation-queue.ts`,
  `tool-context.ts`
- `packages/agent/src/harness/utils/truncate.ts`, `utils/shell-output.ts`;
  `compaction/` — `compaction.ts`, `branch-summarization.ts`, `utils.ts`
- `packages/agent/src/harness/skills.ts`, `prompt-templates.ts`,
  `system-prompt.ts`
- `packages/agent/test/harness/` — `tools.test.ts`, `compaction.test.ts`,
  `truncate.test.ts`, `nodejs-env.test.ts`, `skills.test.ts`,
  `prompt-templates.test.ts`, `resource-formatting.test.ts`,
  `system-prompt.test.ts`, `tool-context.types.ts`

## Anti-Patterns

| Anti-pattern | Why | Evidence |
|---|---|---|
| Importing `node:fs` / `node:child_process` in a tool or util | Breaks the browser bundle and the injected-env contract | `scripts/check-browser-smoke.mjs`; `utils/truncate.ts:51` avoids even `Buffer` |
| Mutating a file without `withFileMutationQueue` | Interleaved read-modify-write between `edit` and `write` on one canonical path | `tools/file-mutation-queue.ts:29`; symlink and abort tests in `tools.test.ts` |
| Hardcoding 2000 / 50KB in a description or check | Diverges from `DEFAULT_MAX_LINES` / `DEFAULT_MAX_BYTES` | `tools/read.ts`, `tools/bash.ts` interpolate the constants |
| Throwing from an `ExecutionEnv` implementation | The interface documents that every failure must be encoded in the `Result` | `FileSystem` doc comment in `types.ts` |
| Cutting compaction between an assistant tool call and its `toolResult` | Produces an invalid provider context | `findValidCutPoints` (`compaction.ts:328`) excludes `toolResult` |
| Reusing the run's `sessionId` or cache retention for a summary request | Poisons provider cache affinity with a throwaway prompt | `completeSimpleWithRetries` (`compaction.ts:118`) |
| Renaming a built-in tool or its `path` argument in isolation | Summary file lists silently go empty | `compaction/utils.ts:24` matches literal names |
| Throwing out of a resource loader | Callers expect a diagnostics pair and treat `not_found` as normal | `skills.ts:49`, `prompt-templates.ts:30` |
