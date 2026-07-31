# Extension API

## When This Applies

Adding or changing extension discovery, the `ExtensionRunner` event dispatch,
the `ExtensionAPI`/`ToolDefinition` surface, or a bundled extension under
`packages/coding-agent/src/extensions/`.

## The Local Pattern

### Two independent discovery implementations exist; only one runs in the CLI

`discoverAndLoadExtensions` (`loader.ts:665`) is a self-contained scanner:
project-local `<cwd>/.pi/extensions/`, then global `<agentDir>/extensions/`,
then explicit `configuredPaths`, de-duplicated by resolved path. It is
exported from `core/index.ts` and `src/index.ts` as part of the public SDK
surface and is exercised heavily by `test/extensions-discovery.test.ts` and
`test/extensions-runner.test.ts`, but nothing under `src/main.ts`,
`src/modes/**`, or `src/core/agent-session-services.ts` calls it — a repo-wide
`grep -rn "discoverAndLoadExtensions(" packages/coding-agent/src` outside
`core/extensions/{loader,index}.ts` and `src/index.ts` returns no call sites.

The CLI runtime instead goes through `DefaultResourceLoader`
(`resource-loader.ts:195`), whose `loadCurrentExtensionSet`
(`resource-loader.ts:548`) resolves paths via `DefaultPackageManager.resolve()`
(`package-manager.ts:885`), which delegates the filesystem scan to
`addAutoDiscoveredResources` (`package-manager.ts:2278`, called at `:934`).
That resolver has its own
precedence — project `.pi/extensions/` first (only when
`settingsManager.isProjectTrusted()`, `package-manager.ts:2343`), then user
`~/.pi/agent/extensions/` (`package-manager.ts:2396`), both merged with
`settings.json` `extensions[]` overrides and installed `packages[]` sources
— and also drives skills, prompts, and themes through the same
`addResources` helper. Treat `discoverAndLoadExtensions` as SDK
surface and a test fixture, not as documentation for what the shipped CLI
does; when changing discovery precedence, `package-manager.ts` is the file
that matters.

### A fixed set of virtual modules, not `node_modules`, backs extension imports

`loader.ts` statically imports every package extensions are allowed to use
(`pi-agent-core`, `pi-tui`, `pi-ai` compat/oauth/providers, `typebox`, the
coding-agent package itself) into `VIRTUAL_MODULES` (`loader.ts:50-74`), and
maps both the current `@earendil-works/*` and legacy `@mariozechner/*`
package names to the same bundled modules. The comment explains why: "These
MUST be static so Bun bundles them into the compiled binary. The
virtualModules option then makes them available to extensions." Node/tsx
mode uses a parallel `getAliases()` map instead (`loader.ts:86-142`); the two
lists must stay in sync.

### `tool_call` handlers can short-circuit by returning `{ block: true }`

`ExtensionRunner.emitToolCall` (`runner.ts:932`) iterates every loaded
extension's `tool_call` handlers in registration order, keeps the last
non-undefined result, and returns immediately once a handler returns
`{ block: true, ... }`:

```ts
if (handlerResult) {
	result = handlerResult as ToolCallEventResult;
	if (result.block) {
		return result;
	}
}
```

(`runner.ts:943-948`.) Later extensions do not run after a block.
`examples/extensions/permission-gate.ts` is the canonical usage: it returns
`{ block: true, reason: "..." }` from a `tool_call` handler to veto a
dangerous bash command.

### `ToolDefinition` is the one shape for built-in and extension tools; `defineTool` is a type helper, not a factory

`pi.registerTool(tool: ToolDefinition<...>)` (`types.ts:1246`) takes the exact
same `ToolDefinition` shape documented in `../core/tools.md` for built-in
tools. `defineTool` (`types.ts:509-513`) does no construction — its doc
comment states its only purpose is to "Preserve parameter inference for
standalone tool definitions" when a tool is assigned to a variable or pushed
into an array (`customTools`) instead of passed inline; it is an identity
cast.

### Inline (path-less) extensions get synthetic `<inline:N>` identities

Extensions passed as `extensionFactories` (plain functions, used by the SDK
and by tests) have no filesystem path. `DefaultResourceLoader.loadExtensionFactories`
assigns them a path of `` `<inline:${isNamed ? input.name : index + 1}>` ``
(`resource-loader.ts:955`) — `<inline:1>`, `<inline:2>`, ... for a bare
function, or `<inline:name>` when the factory is passed as
`{ name, factory }`, so diagnostics and `/extensions` listings can still
identify them. `test/suite/regressions/6260-inline-extension-naming.test.ts`
asserts both forms.

## Reference Files

- `packages/coding-agent/src/core/extensions/loader.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/extensions/types.ts`
- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/package-manager.ts`
- `packages/coding-agent/examples/extensions/permission-gate.ts`
- `packages/coding-agent/examples/extensions/README.md`
- `packages/coding-agent/docs/extensions.md`
- `packages/coding-agent/test/extensions-discovery.test.ts`
- `packages/coding-agent/test/extensions-runner.test.ts`
- `packages/coding-agent/test/suite/regressions/6260-inline-extension-naming.test.ts`
- `packages/coding-agent/test/suite/regressions/2835-tools-allowlist-filters-extension-tools.test.ts`
- `packages/coding-agent/test/suite/regressions/3592-no-builtin-tools-keeps-extension-tools.test.ts`

## Anti-Patterns

- Adding a package to `loader.ts` `VIRTUAL_MODULES` without also adding it to
  `getAliases()` (or vice versa) — Bun-binary and Node/tsx extension
  resolution would diverge.
- Registering a tool as a bare object literal in an array or variable
  position without `defineTool`, then debugging why its parameter type
  widened to `unknown` — wrap it in `defineTool` instead.
- Assuming a `tool_call` handler that returns a result without `block: true`
  stops later handlers — only a truthy `block` short-circuits;
  a non-blocking result still lets later extensions run and overwrite it.
- Re-implementing a project-trust check inside an extension instead of
  relying on `DefaultResourceLoader`, which already withholds project-local
  extensions until the project is trusted.
