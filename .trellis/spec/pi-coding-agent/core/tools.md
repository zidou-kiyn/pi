# Tools

## When This Applies

Adding or modifying a built-in tool (`read`, `bash`, `edit`, `write`, `grep`,
`find`, `ls`) under `packages/coding-agent/src/core/tools/`.

## The Local Pattern

### `ToolDefinition` first, `AgentTool` is a thin wrapper

Every tool module exports `create<X>ToolDefinition(cwd, options)` returning a
`ToolDefinition` (schema, prompt metadata, `execute`, `renderCall`,
`renderResult`) and `create<X>Tool(cwd, options)`, which wraps it into an
`AgentTool` via `wrapToolDefinition` (`tool-definition-wrapper.ts:5`):

```ts
export function wrapToolDefinition<TDetails = unknown>(
	definition: ToolDefinition<any, TDetails>,
	ctxFactory?: () => ExtensionContext,
): AgentTool<any, TDetails> {
	return {
		name: definition.name,
		...
		execute: (toolCallId, params, signal, onUpdate, ctx?: ExtensionContext) =>
			definition.execute(toolCallId, params, signal, onUpdate, ctx ?? (ctxFactory?.() as ExtensionContext)),
	};
}
```

`core/tools/index.ts` re-exports both flavors per tool plus aggregate
`createCodingTool(Definition)s` / `createReadOnlyTool(Definition)s` /
`createAllTool(Definition)s` helpers (`edit`, `bash`, `write` are the "coding"
set; `read`, `grep`, `find`, `ls` are the "read-only" set). `wrapToolDefinition`
is also how `core/extensions/types.ts` `defineTool` turns extension tool
definitions into `AgentTool`s — built-in and extension tools share one path.

### Parameters are `typebox` schemas, input types come from `Static<>`

Every tool defines its schema with `Type.Object(...)` from `typebox` and
derives its input type with `Static<typeof schema>`, e.g. `editSchema`
(`edit.ts:44-53`) and `EditToolInput` (`edit.ts:55`). `prepareArguments` is used to normalize
model quirks before validation — `edit.ts` `prepareEditArguments` folds a
legacy top-level `oldText`/`newText` pair (and a stringified `edits` array
some models emit) into the current `edits[]` shape without exposing the
legacy fields in the public schema.

### Pluggable `Operations` decouple a tool from the local filesystem

`edit.ts` defines an `EditOperations` interface (`readFile`, `writeFile`,
`access`) with `defaultEditOperations` backed by `node:fs/promises`, injected
via `EditToolOptions.operations`. `bash.ts` has the equivalent
`BashOperations` / `createLocalBashOperations`. This is how
`examples/extensions/ssh.ts` redirects all built-in tools to a remote host
without touching tool logic — override the operations, not the tool.

### Concurrent writes to one file are serialized, not per-tool-call

`edit.ts` and `write.ts` wrap their filesystem mutation in
`withFileMutationQueue(absolutePath, fn)` (`file-mutation-queue.ts:32`), which
resolves the real path (`realpath`, falling back to the raw resolved path on
`ENOENT`/`ENOTDIR`) and chains operations per resolved path. Two `edit` calls
against the same file from concurrent tool calls run sequentially; two calls
against different files run in parallel.

### `renderCall` / `renderResult` build stateful TUI components, not strings

`edit.ts` `renderCall` keeps a `Box`-derived `EditCallRenderComponent` in
`context.state.callComponent` across re-renders (pending -> computed preview
-> settled/error), and recomputes a diff preview asynchronously via
`computeEditsDiff` only once per distinct `(path, edits)` key
(`getEditCallRenderComponent`, `setEditPreview`, `edit.ts:147-267`). This
mirrors the `renderShell: "self"` contract used by other coding tools: the
tool owns its full render lifecycle instead of the caller diffing plain text.

## Reference Files

- `packages/coding-agent/src/core/tools/index.ts`
- `packages/coding-agent/src/core/tools/edit.ts`
- `packages/coding-agent/src/core/tools/edit-diff.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
- `packages/coding-agent/src/core/tools/file-mutation-queue.ts`
- `packages/coding-agent/examples/extensions/ssh.ts`
- `packages/coding-agent/test/tools.test.ts`
- `packages/coding-agent/test/edit-tool-legacy-input.test.ts`
- `packages/coding-agent/test/edit-tool-no-full-redraw.test.ts`

## Anti-Patterns

- Calling `node:fs` directly inside a tool's `execute` instead of through an
  `Operations` interface: blocks remote-execution extensions like `ssh.ts`
  from overriding the tool.
- Mutating a target file without `withFileMutationQueue`: concurrent tool
  calls (extension tools, retried calls) can interleave reads/writes on the
  same file.
- Returning a plain string from `renderCall`/`renderResult` for a tool that
  needs incremental updates while streaming, instead of a stateful component
  stored on `context.state` — `edit-tool-no-full-redraw.test.ts` exists
  specifically to catch renders that force a full terminal redraw.
- Adding a new tool option surface without an `Options` interface + a
  `default*Operations` constant, breaking the pattern every existing tool
  follows.
