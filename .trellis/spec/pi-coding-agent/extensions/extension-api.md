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

### A package's `pi` manifest is read through one validating reader

`readPiManifest(packageJsonPath)` (`core/pi-manifest.ts:16`) is the only way to
read the `pi` block out of an installed package's `package.json`, and it is
validation, not just parsing: a field is copied into the returned `PiManifest`
(`core/pi-manifest.ts:3` — `extensions`, `skills`, `prompts`, `themes`) only
when it is an array whose every element is a string. Anything else — a bad
JSON file, a non-object `pi`, or a field like `"skills": "./skills"` — is
dropped silently while the *valid* sibling fields still load;
`test/suite/regressions/7187-malformed-package-manifest.test.ts` asserts
exactly that asymmetry. Every read in `package-manager.ts` goes through it
(`:533`, `:2095`, `:2131`, `:2200`); do not re-parse `package.json` for `pi`
resources anywhere else, and do not make a malformed manifest throw — one bad
installed package must not break resource discovery for the rest.

### `registerMarkdownTransformer` is the only text-level render hook

```ts
registerMarkdownTransformer(transformer: MarkdownTransformer): void;
```

(`types.ts:1287`.) `MarkdownTransformer` is
`(markdown: string, context: MarkdownTransformContext) => string`
(`types.ts:1148`), with `context` carrying `messageType`
(`"user" | "assistant" | "assistant-thinking"`), `isStreaming`, and
`availableWidth` (`types.ts:1142`). Unlike `registerMessageRenderer`, which
replaces a whole component, this rewrites the Markdown *source* before the
renderer parses it, so themes, wrapping, and width accounting keep working.

One transformer per extension: the API assigns to
`extension.markdownTransformer` (`loader.ts:296`, field declared at
`types.ts:1689`), so a second call replaces the first. `ExtensionRunner`
collects them across extensions in load order with `getMarkdownTransformers()`
(`runner.ts:589`).

Application is chained and failure-tolerant:
`createMarkdownTransform(messageType, isStreaming, transformers)`
(`markdown-transform.ts:3`) closes over the list and applies them in order,
feeding each transformer the previous one's output, keeping the result only
when it is a string, and swallowing a throwing transformer so the remaining
ones still run. `InteractiveMode`
reads them once per render through `getMarkdownTransformers()`
(`interactive-mode.ts:1817`) and hands them to `UserMessageComponent`
(`user-message.ts:53`) and `AssistantMessageComponent`
(`assistant-message.ts:112`), which set the closure as
`MarkdownOptions.transform` on the `pi-tui` `Markdown` component — the TUI-side
half of this contract is documented in
[`../../pi-tui/components/component-model.md`](../../pi-tui/components/component-model.md).
This hook is interactive-mode only; print and RPC mode never call it.

### Two extensions owning one tool name is a fatal startup error; a shared shortcut is only a warning

These two collisions look symmetric and are not.

**Tools abort startup.** `detectExtensionConflicts`
(`resource-loader.ts:1058-1092`) walks loaded extensions, and the second
extension to register an already-owned tool (or flag) name produces
`Tool "bash" conflicts with <first owner path>`. `addExtensionConflictDiagnostics`
(`resource-loader.ts:626-633`) pushes each conflict onto
`extensionsResult.errors`, `main.ts:735-738` turns those into `error`
diagnostics reading `Failed to load extension "<path>": ...`, and
`main.ts:844-848` prints the `pi -ne` hint and calls `process.exit(1)`. pi does
not start. The comment on `addExtensionConflictDiagnostics` ("Keep all
extensions loaded ... precedence is handled by load order") describes only the
in-memory result object, not the CLI outcome, and
`ExtensionRunner.getAllRegisteredTools` first-wins merge
(`runner.ts:450-461`) is unreachable in the CLI for a duplicated name. Two
installed packages that both override `bash` (e.g. `pi-tool-display` and
`pi-patty-bg-tasks`) therefore make pi unstartable until one of them is
configured to stand down; reordering `packages[]` changes nothing.

**Shortcuts degrade instead.** `ExtensionRunner.resolveShortcuts`
(`runner.ts:495-535`) emits a warning diagnostic and keeps running. A
built-in key listed in `RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS`
(`runner.ts:71-90`) wins and the extension shortcut is skipped; any other
built-in key (e.g. `ctrl+b` for `tui.editor.cursorLeft`,
`pi-tui/src/keybindings.ts:63`) loses to the extension, which still gets the
key and only costs a per-startup `Extension shortcut conflict` warning. The
user-side fix is `keybindings.json`, where a per-action key list **replaces**
the default list rather than extending it (`pi-tui/src/keybindings.ts:196-200`),
so dropping the built-in claim removes the warning.

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
- `packages/coding-agent/src/core/pi-manifest.ts`
- `packages/coding-agent/src/modes/interactive/components/markdown-transform.ts`
- `packages/coding-agent/examples/extensions/permission-gate.ts`
- `packages/coding-agent/examples/extensions/README.md`
- `packages/coding-agent/docs/extensions.md`
- `packages/coding-agent/test/extensions-discovery.test.ts`
- `packages/coding-agent/test/extensions-runner.test.ts`
- `packages/coding-agent/test/suite/regressions/6260-inline-extension-naming.test.ts`
- `packages/coding-agent/test/suite/regressions/2835-tools-allowlist-filters-extension-tools.test.ts`
- `packages/coding-agent/test/suite/regressions/3592-no-builtin-tools-keeps-extension-tools.test.ts`
- `packages/coding-agent/test/suite/regressions/7187-malformed-package-manifest.test.ts`

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
- Parsing a package's `package.json` `pi` block directly instead of calling
  `readPiManifest` — skips the per-field array-of-strings validation and
  reintroduces issue #7187.
- Making `registerMarkdownTransformer` throw to signal "leave this text
  alone" — throws are swallowed; return the input string unchanged instead.
- Registering a Markdown transformer and expecting it in print or RPC output
  — only the interactive transcript applies it.
- Reasoning about a duplicated extension tool name as a precedence problem
  ("the first registration wins, the other is inert") — the CLI exits 1 before
  any precedence applies. Give an overriding tool a config switch to stand
  down, or a name no other extension claims.
- Adding a keybinding id to `RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS` to
  quiet a shortcut warning — that flips the outcome from "extension wins with
  a warning" to "extension shortcut is skipped entirely".
